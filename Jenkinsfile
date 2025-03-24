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
                git branch: 'main', url: 'git@github.com:your-repo/your-project.git'
            }
        }

        stage('Login to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
            }
        }

        stage('Build & Push Image') {
            steps {
                script {
                    sh """
                        docker build -t ${ECR_REPO}:${IMAGE_TAG} frontend/
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Register Task Definition') {
            steps {
                script {
                    def taskDef = readFile(TASK_DEFINITION_FILE)
                    taskDef = taskDef.replaceAll('<IMAGE_PLACEHOLDER>', "${ECR_REPO}:${IMAGE_TAG}")
                    writeFile(file: 'frontend/task-def-updated.json', text: taskDef)

                    def registerTaskOutput = sh(
                        script: "aws ecs register-task-definition --cli-input-json file://frontend/task-def-updated.json --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()

                    def taskArn = registerTaskOutput.find(/"taskDefinitionArn":\s*"([^"]+)"/)  
                    def taskDefName = taskArn?.split(':')[-2]  
                    def taskDefRevision = taskArn?.split(':')[-1]

                    if (!taskDefName || !taskDefRevision) {
                        error("Failed to extract task definition details.")
                    }

                    env.TASK_DEFINITION = "${taskDefName}:${taskDefRevision}"
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --task-definition ${TASK_DEFINITION} --region ${AWS_REGION}"
            }
        }

        stage('Enable Execute Command') {
            steps {
                sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --enable-execute-command --region ${AWS_REGION}"
            }
        }
    }
}
