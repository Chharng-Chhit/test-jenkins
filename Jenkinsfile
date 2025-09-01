pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "672763004565"
        AWS_DEFAULT_REGION = "ap-southeast-1"
        IMAGE_REPO_NAME = "metabase-clickhouse"
        IMAGE_TAG = "latest"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                // This step is optional since you are pulling a public image
                // git 'https://your-git-repository.com/metabase-clickhouse.git'
            }
        }

        stage('Pull and Tag Docker Image') {
            steps {
                script {
                    // Pull the image from Docker Hub
                    sh "docker pull metabase/metabase:latest"
                    // Tag the image for ECR
                    sh "docker tag metabase/metabase:latest ${REPOSITORY_URI}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    // Log in to ECR
                    sh """
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                    """
                    // Push the image to ECR
                    sh "docker push ${REPOSITORY_URI}:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}