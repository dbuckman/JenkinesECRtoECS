pipeline {
    agent {
      kubernetes {
        label 'mypod'
        defaultContainer 'jnlp'
        yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: some-label-value
spec:
  containers:
  - name: awscli
    image: amazon/aws-cli
    command:
      - 'sleep'
      - '999999'
"""
      }
    }

    environment {
      AWS_ROLE_ARN = "arn:aws:iam::189768267137:role/JenkinsPushToECR"
      AWS_WEB_IDENTITY_TOKEN_FILE = credentials('arch-aws-oidc')
      AWS_REGION = "us-east-1"
      AWS_ECS_SERVICE = "dbuckman-app-demo"
      AWS_ECS_TASK_DEFINITION = "dbuckman-taskdefinition"
      AWS_ECS_COMPATIBILITY = 'FARGATE'
      AWS_ECS_NETWORK_MODE = 'awsvpc'
      AWS_ECS_MEMORY = '512'
      AWS_ECS_CLUSTER = "dbuckman-demo"
      AWS_ECS_TASK_DEFINITION_PATH = './container-definition-update-image.json'
      AWS_ECR_URL = "189768267137.dkr.ecr.us-east-1.amazonaws.com/dbuckman-pipelinetest"
      POM_VERSION = "latest"
    }

    stages {
        stage('Deploy in ECS') {
          steps {
            container('awscli') {
                script {
                    updateContainerDefinitionJsonWithImageVersion()
                    sh("/usr/local/bin/aws ecs register-task-definition --region ${AWS_REGION} --family ${AWS_ECS_TASK_DEFINITION} --execution-role-arn ${AWS_ROLE_ARN} --requires-compatibilities ${AWS_ECS_COMPATIBILITY} --network-mode ${AWS_ECS_NETWORK_MODE} --memory ${AWS_ECS_MEMORY} --container-definitions file://${AWS_ECS_TASK_DEFINITION_PATH}")
                    def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition ${AWS_ECS_TASK_DEFINITION} | egrep \"revision\" | tr \"/\" \" \" | awk '{print \$2}' | sed 's/\"\$//'", returnStdout: true)
                    sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:${taskRevision}")
                }
            }
          }
        }
    }
}

def updateContainerDefinitionJsonWithImageVersion() {
    def containerDefinitionJson = readJSON file: AWS_ECS_TASK_DEFINITION_PATH, returnPojo: true
    containerDefinitionJson[0]['image'] = "${AWS_ECR_URL}:${POM_VERSION}".inspect()
    echo "task definiton json: ${containerDefinitionJson}"
    writeJSON file: AWS_ECS_TASK_DEFINITION_PATH, json: containerDefinitionJson
}
