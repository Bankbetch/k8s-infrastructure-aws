
def notify(message) {
  def gitLog = sh(returnStdout: true, script: "git log -1 --oneline", encoding: "UTF-8").trim()
  def token = 'LINE_TOKEN'
  def jobName = env.JOB_NAME + ' - ' + env.BRANCH_NAME
  def buildNo = env.BUILD_NUMBER

  def url = 'https://notify-api.line.me/api/notify'
  def lineMessage = "${jobName} [#${buildNo}] : ${message} ---> ${gitLog}"
  sh "curl ${url} -H 'Authorization: Bearer ${token}' -F 'message=${lineMessage}'"

  def urlDis = 'https://discord.com/api/webhooks/xxx/xxx'
  sh """curl -H 'Content-Type: application/json' -d '{"content":"${lineMessage}"}' -X POST ${urlDis}"""
}

pipeline {
  environment {
    DEFAULT_IMAGE_REGISTRY = 'xxx.dkr.ecr.ap-southeast-1.amazonaws.com'
    IMAGE_REGISTRY = "${DEFAULT_IMAGE_REGISTRY}/xxx"
    WORKSPACE = '/var/jenkins_home/workspace/infra/helm/helm-values'
    HELM_DIR = '/var/jenkins_home/admin/helm'
    KUBE_DIR = '/var/jenkins_home/admin/kubectl'
  }
  options {
    timeout(time: 1, unit: 'HOURS')
  }
  agent any

  stages {
    stage('Authentication ECR') {
      steps {
        script {
          withAWS(region: 'ap-southeast-1', credentials: 'aws-credentials') {
            sh "aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin ${DEFAULT_IMAGE_REGISTRY}"
          }
        }
      }
    }
    stage('Clone Repo') {
      steps {
        checkout scm
      }
    }
    stage('Build Images') {
      steps {
        script {
          sh "docker build --pull --no-cache -t ${IMAGE_REGISTRY}:${env.BRANCH_NAME}-${env.BUILD_ID} -f Dockerfile ."
        }
      }
    }
    stage('Push Images') {
      steps {
        script {
          sh "docker push ${IMAGE_REGISTRY}:${env.BRANCH_NAME}-${env.BUILD_ID}"
          sh "docker rmi ${IMAGE_REGISTRY}:${env.BRANCH_NAME}-${env.BUILD_ID}"
          }
        }
      }
      stage('Deploy to K8S') {
        steps {
          script {
            try {
              withAWS(region: 'ap-southeast-1', credentials: 'aws-credentials') {
                withKubeConfig([credentialsId: "kube-credentials"]) {
                  def values_file = "${WORKSPACE}/${env.BRANCH_NAME}-values.yaml"
                    sh "helm upgrade -f ${values_file} dev ${HELM_DIR} --set-string app.image=${IMAGE_REGISTRY} --set-string app.tag=${env.BRANCH_NAME}-${env.BUILD_ID} -n ${env.BRANCH_NAME}"
                    sh "${KUBE_DIR} get pod -n ${env.BRANCH_NAME} -o wide"
                    notify("Build successful")
                }
              }
          }catch (Exception e) {
            notify("${e.getMessage()}")
            throw e
          }
        }
      }
    }
  }
}