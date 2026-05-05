pipeline {
    agent any

    environment {
        DOCKERHUB_USER    = 'tusharyadaav'
        BACKEND_IMAGE     = "${DOCKERHUB_USER}/crud-backend"
        FRONTEND_IMAGE    = "${DOCKERHUB_USER}/crud-frontend"
        EKS_CLUSTER       = 'emptrack-cluster'
        AWS_REGION        = 'us-east-1'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Pulling code from GitHub...'
                checkout scm
            }
        }

        stage('Build Backend') {
            steps {
                echo '🔨 Building Backend Docker image...'
                sh """
                    docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} ./backend
                    docker tag ${BACKEND_IMAGE}:${BUILD_NUMBER} ${BACKEND_IMAGE}:latest
                """
            }
        }

        stage('Build Frontend') {
            steps {
                echo '🔨 Building Frontend Docker image...'
                sh """
                    docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} ./frontend
                    docker tag ${FRONTEND_IMAGE}:${BUILD_NUMBER} ${FRONTEND_IMAGE}:latest
                """
            }
        }

        stage('Push Images') {
            steps {
                echo '📤 Pushing images to Docker Hub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${FRONTEND_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                echo '🚀 Deploying to EKS...'
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER}

                        # Update image with build number for versioning
                        kubectl set image deployment/backend \
                          backend=${BACKEND_IMAGE}:${BUILD_NUMBER}

                        kubectl set image deployment/frontend \
                          frontend=${FRONTEND_IMAGE}:${BUILD_NUMBER}

                        # Wait for rollout
                        kubectl rollout status deployment/backend
                        kubectl rollout status deployment/frontend
                    """
                }
            }
        }

    }

    post {
        success {
            echo '✅ Pipeline succeeded! App deployed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check logs above.'
        }
    }
}
