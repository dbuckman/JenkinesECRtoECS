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
      AWS_ROLE_ECS_ARN = "arn:aws:iam::189768267137:role/ecsTaskExecutionRole"
      AWS_ROLE_ARN = "arn:aws:iam::189768267137:role/JenkinsPushToECR"
      AWS_WEB_IDENTITY_TOKEN_FILE = credentials('arch-aws-oidc')
      AWS_REGION = "us-east-1"
      AWS_ECS_SERVICE = "demo-service"
      AWS_ECS_TASK_DEFINITION = "sample-app"
      AWS_ECS_COMPATIBILITY = 'FARGATE'
      AWS_ECS_NETWORK_MODE = 'awsvpc'
      AWS_ECS_MEMORY = '512'
      AWS_ECS_CLUSTER = "dbuckman-cluster"
      AWS_ECS_TASK_DEFINITION_PATH = './container-definition-update-image.json'
      AWS_ECR_URL = "189768267137.dkr.ecr.us-east-1.amazonaws.com/dbuckman-pipelinetest"
      AWS_ECR_IMAGE = "dbuckman-pipelinetest"
      CONFIG_DATA = """{
    "family": "sample-app",
    "networkMode": "awsvpc",
    "containerDefinitions": [
        {
            "name": "demo-app",
            "image": "189768267137.dkr.ecr.us-east-1.amazonaws.com/dbuckman-pipelinetest:latest",
            "portMappings": [
                {
                    "containerPort": 8080,
                    "hostPort": 8080,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "entryPoint": [
                "java",
                "-Djava.security.egd=file:/dev/./urandom",
                "-jar",
                "/app.jar"
            ]
        }
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "256",
    "memory": "512"
}"""
    }

    stages {
        stage('Deploy in ECS') {
          steps {
            container('awscli') { 
                script {
                    sh("yum install -y less groff jq")
                    def ECR_VERSION1 = sh(script: "/usr/local/bin/aws ecr describe-images --region ${AWS_REGION} --repository-name ${AWS_ECR_IMAGE} --output text --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]'", returnStdout: true)
                    def ECR_VERSION2 = sh(script: "/usr/bin/echo ${ECR_VERSION1} | tr '\t' '\n'", returnStdout: true)
                    def ECR_VERSION = sh(script: "/usr/bin/echo ${ECR_VERSION2} | tail -1", returnStdout: true)
                    sh("cat > task.json <<- EOM\n${CONFIG_DATA}\nEOM")
                    sh("jq '.containerDefinitions[0].image = \"${AWS_ECR_URL}:${ECR_VERSION}\"' task.json > task2.json")
                    sh("mv task2.json task.json")
                    sh("cat task.json")

                    sh("/usr/local/bin/aws ecs register-task-definition --execution-role-arn ${AWS_ROLE_ECS_ARN} --cli-input-json file://task.json")
                    def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition sample-app | egrep \"revision\" | awk '{print \$2}' | sed 's/,//g'", returnStdout: true)
                    
                    sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:${taskRevision}")
                }
            }
          }
        }
    }
}
