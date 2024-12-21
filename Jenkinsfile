pipeline {
    agent {
        label 'MacLocal'
    }
    parameters {
        string(name: 'branch', defaultValue: 'main', description: 'Git branch to build')
        string(name: 'environment', defaultValue: 'dev', description: 'Environment to deploy')
    }
    environment {
        PORT = '3001'
        DOCKER_IMAGE = 'todo-app'
        SLACK_CHANNEL = '#all-dify-bot-demo'
    }
    stages {
        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.branch}"
                checkout([
                    $class: 'GitSCM', 
                    branches: [[name: "*/${params.branch}"]],
                    userRemoteConfigs: [[url: 'https://github.com/Danielyan86/DevOpsGPT_Demo.git']]
                ])
            }
        }
        stage('Build Docker Image') {
            steps {
                echo "Building for environment: ${params.environment}"
                sh """
                    echo "Building Docker image..."
                    docker build -t ${DOCKER_IMAGE}:${params.environment} .
                    docker tag ${DOCKER_IMAGE}:${params.environment} ${DOCKER_IMAGE}:latest
                """
            }
        }
        stage('Deploy') {
            steps {
                sh """
                    echo "Stopping existing container if any..."
                    docker stop ${DOCKER_IMAGE} || true
                    docker rm ${DOCKER_IMAGE} || true
                    
                    echo "Starting new container..."
                    docker run -d \\
                        --name ${DOCKER_IMAGE} \\
                        -p ${PORT}:3001 \\
                        ${DOCKER_IMAGE}:latest
                    
                    echo "Waiting for server to start..."
                    sleep 10
                    
                    echo "Checking if server is running..."
                    curl -f http://localhost:${PORT} || {
                        echo "Server failed to start. Checking container logs:"
                        docker logs ${DOCKER_IMAGE}
                        exit 1
                    }
                """
            }
        }
    }
    post {
        always {
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def buildDuration = currentBuild.durationString
                
                echo "Build completed with status: ${buildStatus}"
                echo "Build duration: ${buildDuration}"
                
                def message = """
                    *Build Status*: ${buildStatus}
                    *Job*: ${env.JOB_NAME}
                    *Build Number*: #${env.BUILD_NUMBER}
                    *Branch*: ${params.branch}
                    *Environment*: ${params.environment}
                    *Duration*: ${buildDuration}
                    *Build URL*: ${env.BUILD_URL}
                    *Image Tag*: ${DOCKER_IMAGE}:${params.environment}
                    *Deployed Port*: ${PORT}
                """
                
                echo "Sending notification to Slack channel: ${SLACK_CHANNEL}"
                try {
                    slackSend(
                        tokenCredentialId: 'SlackToken',
                        channel: SLACK_CHANNEL,
                        color: buildStatus == 'SUCCESS' ? 'good' : 'danger',
                        message: message,
                        notifyCommitters: true
                    )
                    echo "Slack notification sent successfully"
                } catch (Exception e) {
                    echo "Failed to send Slack notification: ${e.message}"
                    echo "Stack trace: ${e.printStackTrace()}"
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
