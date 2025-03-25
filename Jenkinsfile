pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECS_CLUSTER = 'Techthree-cluster'
        ECS_SERVICE = 'frontend-Service'
        TASK_DEFINITION_FILE = 'ecs-task-definition.json'
        ECR_REPO = '058264111898.dkr.ecr.us-east-1.amazonaws.com/techthree-repo'
        IMAGE_TAG = "frontend-${env.BUILD_NUMBER}"
        FRONTEND_DIR = 'Frontend' // Corrected case to match your repository
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/shahroz899/3-tier-ECS.git'
                // Debug: List directory contents to verify structure
                sh 'ls -la'
            }
        }

        stage('Login to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
            }
        }

        stage('Build & Push Frontend Image') {
            steps {
                dir(FRONTEND_DIR) { // Now using correct case-sensitive directory name
                    sh """
                        docker build -t ${ECR_REPO}:frontend-latest .
                        docker tag ${ECR_REPO}:frontend-latest ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:frontend-latest
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Register Task Definition') {
            steps {
                script {
                    def taskDef = readFile("${FRONTEND_DIR}/${TASK_DEFINITION_FILE}")
                    taskDef = taskDef.replaceAll('<IMAGE_PLACEHOLDER>', "${ECR_REPO}:frontend-latest")
                    writeFile(file: 'task-def-updated.json', text: taskDef)

                    sh "aws ecs register-task-definition --cli-input-json file://task-def-updated.json --region ${AWS_REGION}"
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment --region ${AWS_REGION}"
            }
        }
    }

    post {
        always {
            // Clean up temporary files
            sh 'rm -f task-def-updated.json'
        }
    }
}
