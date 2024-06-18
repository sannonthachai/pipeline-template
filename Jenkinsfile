pipeline {
    agent {
      kubernetes {
        yaml '''
        kind: Pod
        metadata:
          name: jenkins-pipeline
          namespace: app-svcs
        spec:
          nodeSelector:
            worker: pipelineNodeGroup
          containers:
            - name: builder
              image: gcr.io/kaniko-project/executor:debug
              imagePullPolicy: Always
              command:
                - /busybox/cat
              tty: true
              volumeMounts:
                - name: docker-config
                  mountPath: /root/.docker/
                - name: aws-secret
                  mountPath: /root/.aws/
              resources:
                requests:
                  memory: "2.5Gi"
                  cpu: "1200m"
          volumes:
            - name: docker-config
              configMap:
                name: docker-config
            - name: aws-secret
              secret:
                secretName: aws-secret
          '''
      }
    }
    options {
        skipStagesAfterUnstable()
        disableConcurrentBuilds abortPrevious: true 
    }
    environment {
        TAG = "${env.GIT_BRANCH}-${env.GIT_COMMIT}"
        HELM_DIR = "k8s/helm"
        ARGO_REPO_CRED_ID = ""
        ARGO_URL = ""
        ARGO_DIR = ""
        ARGO_RM_DIR = "${env.ARGO_DIR}/helm"
        HELM_VALUE_FILE_PRODUCTION = "${env.ARGO_DIR}/helm/values.yaml"
        HELM_VALUE_FILE = "${env.ARGO_DIR}/helm/values.${env.GIT_BRANCH}.yaml"
        ARGO_HELM_VALUE = "${env.GIT_BRANCH == 'master' ? env.HELM_VALUE_FILE_PRODUCTION : env.HELM_VALUE_FILE}"
        GIT_EMAIL = ""
        GIT_NAME = ""
        JENKINS_WEBHOOK_URL = ""
        BITBUCKET_URL = "https://bitbucket.org/globish/${env.ARGO_DIR}/commits/${env.GIT_COMMIT}"
    }
    stages {
        stage("Install dependencies") {
            when {
                anyOf {
                    branch "dev"
                    branch "staging"
                    branch "master"
                    branch "cronjob"
                }
            }
            steps {
                nodejs("Node-14.0.0") {
                    sh "yarn"
                }
            }
        }
        stage("Test application") {
            when {
                branch "staging"
            }
            steps {
                nodejs("Node-14.0.0") {
                    sh "yarn run test"
                }
            }
        }
        stage("Build application") {
            when {
                anyOf {
                    branch "dev"
                    branch "staging"
                    branch "master"
                    branch "cronjob"
                }
            }
            steps {
                nodejs("Node-14.0.0") {
                    sh "yarn run build"
                }
            }
        }
        stage("Build docker image") {
            when {
                anyOf {
                    branch "dev"
                    branch "staging"
                    branch "master"
                    branch "cronjob"
                }
            }
            steps {
                container("builder") {
                    script {
                        sh "/kaniko/executor --context `pwd` --dockerfile ./Dockerfile.prod --destination 248783859424.dkr.ecr.ap-southeast-1.amazonaws.com/${env.ARGO_DIR}:${env.TAG}"
                    }
                }
            }
        }
        stage("Publish helm chart") {
            when {
                anyOf {
                    branch "dev"
                    branch "staging"
                    branch "master"
                    branch "cronjob"
                }
            }
            steps {
                sh "cp -r ${env.HELM_DIR} .."
                git branch: "${env.GIT_BRANCH}",
                    credentialsId: "${env.ARGO_REPO_CRED_ID}",
                    url: "${env.ARGO_URL}"
                sh "git config --global user.email ${env.GIT_EMAIL}"
                sh "git config --global user.name ${env.GIT_NAME}"
                sh "rm -rf ${env.ARGO_RM_DIR}"
                sh "mv ../helm ${env.ARGO_DIR}"
                sh "sed -i 's/latest/${env.TAG}/g' ${env.ARGO_HELM_VALUE}"
                sh "git add ${env.ARGO_DIR}"
                sh "git commit -m ${env.TAG}"
                withCredentials([gitUsernamePassword(credentialsId: "${env.ARGO_REPO_CRED_ID}", gitToolName: "Default")]) {
                    sh "git push -u origin ${env.GIT_BRANCH}"
                }
            }
        }
    }
    post {
      success {
        script {
            if (["dev","staging","master", "cronjob"].contains(env.GIT_BRANCH) == true) {
              googlechatnotification url: "${env.JENKINS_WEBHOOK_URL}", message: getMessage(true), messageFormat: "card"
            }
        }
      }
      failure {
        script {
          if (["dev","staging","master", "cronjob"].contains(env.GIT_BRANCH) == true) {
            googlechatnotification url: "${env.JENKINS_WEBHOOK_URL}", message: getMessage(false), messageFormat: "card"
          }
        }
      }
      always {
        cleanWs(cleanWhenNotBuilt: false,
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true,
                patterns: [[pattern: '.gitignore', type: 'INCLUDE'],
                           [pattern: '.propsfile', type: 'EXCLUDE']])
      }
    }
}

def getMessage(Boolean success){
  def status = success ? "<font color='#80e27e'>SUCCESS</font>" : "<font color='#FF0000'>FAILED</font>"
  def message = """
  {
    "cardsV2": [
      {
        "cardId": "unique-card-id",
        "card": {
          "header": {
            "title": "Jenkins Bot Notification",
          },
          "sections": [
            {
              "widgets": [
                {
                  "decoratedText": {
                    "topLabel": "Project",
                    "text": "${env.ARGO_DIR}"
                  }
                },
                {
                  "decoratedText": {
                    "topLabel": "Branch",
                    "text": "${env.GIT_BRANCH}"
                  }
                },
                {
                  "decoratedText": {
                    "topLabel": "Commit Id",
                    "text": "${env.GIT_COMMIT[0..6]}"
                  }
                },
                {
                  "decoratedText": {
                    "topLabel": "Build Number",
                    "text": "${env.BUILD_NUMBER}"
                  }
                },
                {
                  "decoratedText": {
                    "topLabel": "Build URL",
                    "text": "<a href='${env.BITBUCKET_URL}'>${env.BITBUCKET_URL}</a>"
                  }
                },
                {
                  "decoratedText": {
                    "topLabel": "Status",
                    "text": "${status}"
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
  """
  return message;
}