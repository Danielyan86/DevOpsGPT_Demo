pipeline {
    agent any
    environment {
        PORT = '3001'
        DOCKER_IMAGE = 'todo-app'
    }
    stages {
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                    echo "Stopping existing container if any..."
                    docker stop ${DOCKER_IMAGE} || true
                    docker rm ${DOCKER_IMAGE} || true
                    
                    echo "Starting new container..."
                    docker run -d \
                        --name ${DOCKER_IMAGE} \
                        -p ${PORT}:3001 \
                        ${DOCKER_IMAGE}:latest
                    
                    echo "Waiting for server to start..."
                    sleep 10
                    
                    echo "Checking if server is running..."
                    curl -f http://localhost:${PORT} || {
                        echo "Server failed to start. Checking container logs:"
                        docker logs ${DOCKER_IMAGE}
                        exit 1
                    }
                '''
            }
        }
    }
    post {
        failure {
            sh '''
                echo "Deployment failed, cleaning up..."
                docker stop ${DOCKER_IMAGE} || true
                docker rm ${DOCKER_IMAGE} || true
            '''
        }
        success {
            echo "Application successfully deployed in Docker on port ${PORT}"
        }
    }
} 