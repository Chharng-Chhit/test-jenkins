pipeline {
    agent none  // No global agent, each stage will define its own
    environment {
        DOCKER_CONFIG = '/tmp/.docker'  // Set to a directory with write access
        repoUri = "672763004565.dkr.ecr.ap-southeast-1.amazonaws.com/webform"
        repoRegistryUrl = "https://672763004565.dkr.ecr.ap-southeast-1.amazonaws.com"
        registryCreds = 'ecr:ap-southeast-1:awscreds'
        cluster = "webform"
        service = "webform-svc"
        region = 'ap-southeast-1'
    }

    stages {
        stage('Docker Test') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    sh 'docker ps'
                }
            }
        }

        stage('Build Docker Image') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo 'Building Docker Image from Dockerfile...'
                    sh 'mkdir -p /tmp/.docker'  // Ensure the directory exists
                    dockerImage = docker.build(repoUri + ":$BUILD_NUMBER")
                }
            }
        }

        stage('Push Docker Image to ECR') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo "Pushing Docker Image to ECR..."
                    docker.withRegistry(repoRegistryUrl, registryCreds) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint=""'
                }
            }
            steps {
                script {
                    echo "Deploying Image to ECS..."
                    withAWS(credentials: 'awscreds', region: "${region}") {
                        // Check if cluster exists, create if not
                        def clusterExists = sh(
                            script: "aws ecs describe-clusters --clusters ${cluster} --region ${region} | grep '\"status\": \"ACTIVE\"'",
                            returnStatus: true
                        )
                        if (clusterExists != 0) {
                            echo "Cluster '${cluster}' does not exist. Creating..."
                            sh "aws ecs create-cluster --cluster-name ${cluster} --region ${region}"
                        } else {
                            echo "Cluster '${cluster}' exists."
                        }

                        // Check if task definition exists, create default if not
                        def taskDefName = "${service}-task"
                        def taskDefExists = sh(
                            script: "aws ecs describe-task-definition --task-definition ${taskDefName} --region ${region}",
                            returnStatus: true
                        )
                        if (taskDefExists != 0) {
                            echo "Task definition '${taskDefName}' does not exist. Creating default..."
                            sh """
                            aws ecs register-task-definition \
                                --family ${taskDefName} \
                                --network-mode awsvpc \
                                --requires-compatibilities FARGATE \
                                --cpu '256' \
                                --memory '512' \
                                --execution-role-arn arn:aws:iam::${repoUri.split('\\.')[0]}:role/ecsTaskExecutionRole \
                                --container-definitions '[
                                    {
                                        "name": "${service}",
                                        "image": "${repoRegistryUrl}/webform:latest",
                                        "essential": true,
                                        "portMappings": [
                                            {
                                                "containerPort": 80,
                                                "hostPort": 80,
                                                "protocol": "tcp"
                                            }
                                        ]
                                    }
                                ]' \
                                --region ${region}
                            """
                        } else {
                            echo "Task definition '${taskDefName}' exists."
                        }

                        // Update service
                        sh "aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment --region ${region}"
                    }
                }
            }
        }
    }
}