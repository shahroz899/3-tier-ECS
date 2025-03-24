pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID = "058264111898"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/techthree-repo"
        FRONTEND_IMAGE = "${ECR_REPO}"
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
                        script: "aws ssm get-parameter --name /ecs/frontend/config --query Parameter.Value --output text --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Backup Existing Image') {
            steps {
                script {
                    def imageDigest = sh(
                        script: "aws ecr list-images --repository-name techthree-repo --region $AWS_REGION --query 'imageIds[?imageTag==\`latest\`].imageDigest' --output text",
                        returnStdout: true
                    ).trim()

                    if (imageDigest) {
                        def timestamp = sh(
                            script: "date +%Y%m%d%H%M%S",
                            returnStdout: true
                        ).trim()

                        def backupTag = "backup-${timestamp}"

                        sh """
                        # Fetch the image manifest and save to a file
                        aws ecr batch-get-image --repository-name techthree-repo --region $AWS_REGION --image-ids imageDigest=${imageDigest} --query 'images[].imageManifest' --output text > image-manifest.json
                        
                        # Push the same image under a backup tag
                        aws ecr put-image --repository-name techthree-repo --region $AWS_REGION --image-tag ${backupTag} --image-manifest file://image-manifest.json
                        """
                    } else {
                        echo "No existing 'latest' image found, skipping backup."
                    }
                }
            }
        }

        stage('Build & Push Frontend') {
            steps {
                sh '''
                docker build -f Frontend/Dockerfile --build-arg REACT_APP_PUBLIC_URL=${BACKEND_URL} -t ${FRONTEND_IMAGE}:latest Frontend/
                docker push ${FRONTEND_IMAGE}:latest
                '''
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh '''
                aws ecs update-service --region us-east-1 --cluster Techthree-cluster --service frontend-service --force-new-deployment
                '''
            }
        }
    }
}
