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
    }

    stages {
        stage('Build Nodejs Container') {
            steps {
                sh "docker build -t $IMAGE_REPO_NAME:latest ."
            }
        }

        stage('Simple Test Nodejs Container') {
            steps {
                sh 'ls -la'
                sh "docker stop \$(docker ps -q)"
                sh 'docker run --rm -d -p 3000:3000 $IMAGE_REPO_NAME:latest'
                sh 'sleep 5'
                sh 'if curl -I http://localhost:3000 | grep -q "200 OK"; then echo "Test passed"; else echo "Test failed"; exit 1; fi'
            }
        }
        stage('Push Nodejs Container to ECR') {
            steps {
                script {
                    def dockerLogin = 'aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com'
                    sh dockerLogin

                    sh "docker tag $IMAGE_REPO_NAME:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:latest"
                
                    sh "docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:latest"
                }
            }
        }

        post {
            always {
                sh 'docker kill \$(docker ps -q)'
            }
            success {
                echo 'Successfully built and pushed the docker image'
            }
            failure {
                echo 'There was some failure in the code'
            }
        }
    }
}