pipeline {
    agent {
        label 'MacLocal'
    }
    environment {
        PORT = '3001'
        DOCKER_IMAGE = 'todo-app'
        SLACK_CHANNEL = '#all-dify-bot-demo'
    }
    def sendSlackMessage(String message, String color = 'good') {
        try {
            slackSend(
                tokenCredentialId: 'SlackToken',
                channel: SLACK_CHANNEL,
                color: color,
                message: message
            )
        } catch (Exception e) {
            echo "Failed to send Slack notification: ${e.message}"
        }
    }
    stages {
        stage('Initialize') {
            steps {
                script {
                    // Define the Slack message function
                    env.sendSlackMessage = { message, color = 'good' ->
                        try {
                            slackSend(
                                tokenCredentialId: 'SlackToken',
                                channel: SLACK_CHANNEL,
                                color: color,
                                message: message
                            )
                        } catch (Exception e) {
                            echo "Failed to send Slack notification: ${e.message}"
                        }
                    }
                }
            }
        }
        stage('Notify Start') {
            steps {
                script {
                    env.sendSlackMessage("""
                        :rocket: *Build Started* :hammer_and_wrench:
                        *Job*: ${env.JOB_NAME}
                        *Build Number*: #${env.BUILD_NUMBER}
                        *Status*: :building_construction: Building...
                        *Build URL*: ${env.BUILD_URL}
                    """, 'warning')
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${DOCKER_IMAGE}:latest .
                '''
                script {
                    env.sendSlackMessage("""
                        :whale: *Docker Image Built Successfully* :package:
                        *Job*: ${env.JOB_NAME}
                        *Image*: ${DOCKER_IMAGE}:latest
                    """)
                }
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
                script {
                    env.sendSlackMessage("""
                        :white_check_mark: *Deployment Successful* :tada:
                        *Job*: ${env.JOB_NAME}
                        *Build Number*: #${env.BUILD_NUMBER}
                        *Status*: :green_circle: Running
                        *Application URL*: http://localhost:${PORT}
                        *Build URL*: ${env.BUILD_URL}
                    """)
                }
            }
        }
    }
    post {
        always {
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def buildDuration = currentBuild.durationString
                
                def message = """
                    :clipboard: *Build Summary* :mag:
                    *Build Status*: ${buildStatus == 'SUCCESS' ? ':large_green_circle:' : ':red_circle:'} ${buildStatus}
                    *Job*: ${env.JOB_NAME}
                    *Build Number*: #${env.BUILD_NUMBER}
                    *Duration*: :hourglass_flowing_sand: ${buildDuration}
                    *Build URL*: ${env.BUILD_URL}
                    *Image Tag*: :whale: ${DOCKER_IMAGE}:latest
                    *Deployed Port*: :computer: ${PORT}
                """
                
                env.sendSlackMessage(
                    message, 
                    buildStatus == 'SUCCESS' ? 'good' : 'danger'
                )
            }
        }
        failure {
            sh '''
                echo "Deployment failed, cleaning up..."
                docker stop ${DOCKER_IMAGE} || true
                docker rm ${DOCKER_IMAGE} || true
            '''
            script {
                env.sendSlackMessage("""
                    :x: *Deployment Failed* :boom:
                    *Job*: ${env.JOB_NAME}
                    *Build Number*: #${env.BUILD_NUMBER}
                    *Status*: :red_circle: Failed
                    *Build URL*: ${env.BUILD_URL}
                    :sos: Check logs for details
                """, 'danger')
            }
        }
        success {
            echo "Application successfully deployed in Docker on port ${PORT}"
        }
    }
}
