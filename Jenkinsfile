pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        APPLICATION_NAME   = 'nodejs-app'
        IMAGE_REPO_NAME    = 'nodejs-k8s'
        IMAGE_TAG          = 'latest'
        EKS_CLUSTER_NAME   = 'node-cluster-k8s'
        AWS_ACCOUNT_ID     = credentials('aws-account-id')
        AWS_CREDENTIALS    = credentials('my-aws-credentials')
        REPOSITORY_URI     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
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
                    withCredentials([usernamePassword(credentialsId: 'my-aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}"
                    }
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
                    withCredentials([usernamePassword(credentialsId: 'my-aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "docker tag $IMAGE_REPO_NAME:$IMAGE_TAG ${REPOSITORY_URI}/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                        sh "docker push ${REPOSITORY_URI}/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'my-aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            aws eks --region ${AWS_DEFAULT_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}
                            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | kubectl create secret docker-registry aws-ecr-secret --docker-server=${REPOSITORY_URI} --docker-username=AWS --docker-password=$(aws ecr get-login-password --region ${AWS_DEFAULT_REGION})
                            kubectl apply -f k8s/
                            kubectl rollout restart deployment/nodejs-deployment
                        """
                    }
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