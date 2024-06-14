pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        APPLICATION_NAME   = 'nodejs-app'
        AWS_DEFAULT_REGION = 'us-east-1'
        IMAGE_REPO_NAME    = 'nodejs-k8s'
        AWS_ACCOUNT_ID     = credentials('aws-account-id')
        IMAGE_TAG          = 'latest'
        REPOSITORY_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to AWS ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 312211201134.dkr.ecr.us-east-1.amazonaws.com
                    """
                }
            }
        }

        stage('Build Nodejs Container') {
            steps {
                sh "docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG ."
            }
        }

        stage('Simple Test Nodejs Container') {
            steps {
                script {
                    def runningContainers = sh(script: "docker ps -q", returnStdout: true).trim()
                    if (runningContainers) {
                        sh "docker stop $runningContainers"
                    }
                    sh "docker run --rm -d -p 3000:3000 $IMAGE_REPO_NAME:$IMAGE_TAG"
                    sh "sleep 5"
                    sh """
                        if curl -I http://localhost:3000 | grep -q "200 OK"; then 
                            echo "Test passed"; 
                        else 
                            echo "Test failed"; 
                            exit 1; 
                        fi
                    """
                }
            }
        }

        stage('Push Nodejs Container to ECR') {
            steps {
                script {
                    sh """
                        docker tag nodejs-k8s:latest 312211201134.dkr.ecr.us-east-1.amazonaws.com/nodejs-k8s:latest
                    """
                    sh """
                        docker push 312211201134.dkr.ecr.us-east-1.amazonaws.com/nodejs-k8s:latest
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
            sh 'docker stop $(docker ps -a -q)'
        }
        success {
            echo 'Successfully built and pushed the docker image'
        }
        failure {
            echo 'There was some failure in the code'
        }
    }
}
