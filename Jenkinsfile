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
      AWS_ECS_SERVICE = "demo-service"
      AWS_ECS_TASK_DEFINITION = "sample-app"
      AWS_ECS_COMPATIBILITY = 'FARGATE'
      AWS_ECS_NETWORK_MODE = 'awsvpc'
      AWS_ECS_MEMORY = '512'
      AWS_ECS_CLUSTER = "dbuckman-demo"
      AWS_ECS_TASK_DEFINITION_PATH = './container-definition-update-image.json'
      AWS_ECR_URL = "189768267137.dkr.ecr.us-east-1.amazonaws.com/dbuckman-pipelinetest"
      POM_VERSION = "latest"
        CONFIG_DATA = """[{
                    "portMappings": [
                        {
                         "hostPort": 8080,
                         "protocol": "tcp",
                         "containerPort": 8080
                        }
                    ],
                    "cpu": 256,
                    "name": "dbuckmanDemo",
                    "image": "189768267137.dkr.ecr.us-east-1.amazonaws.com/dbuckman-pipelinetest:latest"
                }]"""
    }

    stages {
        stage('Deploy in ECS') {
          steps {
            container('awscli') { 
                script {
                    sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:2")
                }
            }
          }
        }
    }
}
