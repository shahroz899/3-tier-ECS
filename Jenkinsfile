pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECS_CLUSTER = 'Techthree-cluster'
        ECS_SERVICE = 'frontend-Service'
        TASK_DEFINITION_FILE = 'ecs-task-definition.json'
        ECR_REPO = '134819843120.dkr.ecr.us-east-1.amazonaws.com/techthree-repo'
        IMAGE_TAG = "frontend-${env.BUILD_NUMBER}"
        FRONTEND_DIR = 'Frontend'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/shahroz899/3-tier-ECS.git'
                sh 'ls -la'
            }
        }

        stage('Login to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
            }
        }

        stage('Retrieve Backend URL from Parameter Store') {
            steps {
                script {
                    env.REACT_APP_PUBLIC_URL = sh(
                        script: "aws ssm get-parameter --name '/ecs/frontend/backendurl' --region ${AWS_REGION} --query Parameter.Value --output text",
                        returnStdout: true
                    ).trim()
                    echo "Retrieved REACT_APP_PUBLIC_URL: ${env.REACT_APP_PUBLIC_URL}"
                }
            }
        }

        stage('Build & Push Frontend Image') {
            steps {
                dir(FRONTEND_DIR) {
                    sh """
                        docker build \
                            --build-arg REACT_APP_PUBLIC_URL=\"${env.REACT_APP_PUBLIC_URL}\" \
                            -t ${ECR_REPO}:frontend-latest .
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
                    // Read and update task definition file
                    def taskDef = readFile("${FRONTEND_DIR}/${TASK_DEFINITION_FILE}")
                    taskDef = taskDef.replaceAll('<IMAGE_PLACEHOLDER>', "${ECR_REPO}:frontend-latest")
                    writeFile(file: 'task-def-updated.json', text: taskDef)

                    // Register task definition and capture ARN
                    def registerOutput = sh(
                        script: "aws ecs register-task-definition --cli-input-json file://task-def-updated.json --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()
                    
                    // Parse the task definition ARN from output
                    def taskDefJson = readJSON(text: registerOutput)
                    env.LATEST_TASK_DEF_ARN = taskDefJson.taskDefinition.taskDefinitionArn
                    echo "Registered Task Definition ARN: ${env.LATEST_TASK_DEF_ARN}"
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                script {
                    // Update service with explicit task definition ARN
                    sh """
                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --task-definition ${env.LATEST_TASK_DEF_ARN} \
                            --force-new-deployment \
                            --region ${AWS_REGION}
                    """

                    // Wait for service to stabilize
                    sh """
                        aws ecs wait services-stable \
                            --cluster ${ECS_CLUSTER} \
                            --services ${ECS_SERVICE} \
                            --region ${AWS_REGION}
                    """
                    echo "Service updated successfully with new task definition"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    // Verify the running task definition
                    def serviceInfo = sh(
                        script: "aws ecs describe-services --cluster ${ECS_CLUSTER} --services ${ECS_SERVICE} --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()
                    
                    def serviceJson = readJSON(text: serviceInfo)
                    def runningTaskDef = serviceJson.services[0].taskDefinition
                    echo "Currently running Task Definition: ${runningTaskDef}"
                    
                    if (!runningTaskDef.contains(env.LATEST_TASK_DEF_ARN)) {
                        error("Service did not update to the latest task definition!")
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f task-def-updated.json'
            echo "Pipeline execution completed"
        }
        failure {
            echo "Pipeline failed - check logs for details"
        }
    }
}
