pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECS_CLUSTER = 'Techthree-cluster'
        ECS_SERVICE = 'frontend-Service'
        TASK_DEFINITION_FILE = 'frontend/ecs-task-definition.json'
        ECR_REPO = '058264111898.dkr.ecr.us-east-1.amazonaws.com/techthree-repo'
        IMAGE_TAG = "frontend-${env.BUILD_NUMBER}"  // Unique tag per build
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your-repo/your-project.git'
            }
        }

        stage('Login to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
            }
        }

        stage('Build & Push Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO}:latest frontend/
                    docker push ${ECR_REPO}:latest
                """
            }
        }

        stage('Register Task Definition') {
            steps {
                script {
                    def taskDef = readFile(TASK_DEFINITION_FILE)
                    taskDef = taskDef.replaceAll('<IMAGE_PLACEHOLDER>', "${ECR_REPO}:latest")
                    writeFile(file: 'frontend/task-def-updated.json', text: taskDef)

                    sh "aws ecs register-task-definition --cli-input-json file://frontend/task-def-updated.json --region ${AWS_REGION}"
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment --region ${AWS_REGION}"
            }
        }

        stage('Enable Execute Command') {
            steps {
                sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --enable-execute-command --region ${AWS_REGION}"
            }
        }
    }
}
