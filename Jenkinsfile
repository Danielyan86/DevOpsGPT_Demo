pipeline {
    agent {
        label 'MacLocal'
    }
    environment {
        PORT = '3001'
        DOCKER_IMAGE = 'todo-app'
        SLACK_CHANNEL = '#jenkins-notifications'
        SLACK_TOKEN = credentials('SlackToken')
        SLACK_TEAM_DOMAIN = 'difybotdemo'
        SLACK_BASE_URL = 'https://hooks.slack.com/services/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'
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
        always {
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def buildDuration = currentBuild.durationString
                
                def message = """
                    *Build Status*: ${buildStatus}
                    *Job*: ${env.JOB_NAME}
                    *Build Number*: #${env.BUILD_NUMBER}
                    *Duration*: ${buildDuration}
                    *Build URL*: ${env.BUILD_URL}
                """
                
                withCredentials([string(credentialsId: 'SlackToken', variable: 'SLACK_TOKEN')]) {
                    slackSend(
                        channel: SLACK_CHANNEL,
                        token: SLACK_TOKEN,
                        color: buildStatus == 'SUCCESS' ? 'good' : 'danger',
                        message: message
                    )
                }
            }
        }
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