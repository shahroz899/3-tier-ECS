pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID = "058264111898"
        FRONTEND_IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/techthree-repo/frontend"
    }

    stages {
        stage('Login to AWS ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Fetch Env Variables') {
            steps {
                script {
                    env.BACKEND_URL = sh(
                        script: "aws ssm get-parameter --name /ecs/frontend/config --query Parameter.Value --output text",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build & Push Frontend') {
            steps {
                sh '''
                docker build --build-arg REACT_APP_PUBLIC_URL=${BACKEND_URL} -t ${FRONTEND_IMAGE}:latest .
                docker push ${FRONTEND_IMAGE}:latest
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                aws ecs update-service --cluster Techthree-cluster --service frontend-service --force-new-deployment
                '''
            }
        }
    }
}
