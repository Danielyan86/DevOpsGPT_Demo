pipeline {
    agent {
        label 'MacLocal'
    }
    parameters {
        string(name: 'branch', defaultValue: 'main', description: 'Git branch to build')
        string(name: 'environment', defaultValue: 'dev', description: 'Environment to deploy')
        string(name: 'SLACK_CHANNEL', defaultValue: '#chatops', description: 'Slack channel for notifications')
    }
    environment {
        PORT = '3001'
        DOCKER_IMAGE = 'todo-app'
        TOTAL_STAGES = '3'
    }
    def generateProgressBar(currentStage) {
        script {
            def percentage = (currentStage / TOTAL_STAGES.toInteger()) * 100
            def progressBar = ""
            def barLength = 20
            def filledLength = (percentage / 100 * barLength).toInteger()
            
            progressBar = "[${"▓" * filledLength}${"░" * (barLength - filledLength)}] ${percentage.intValue()}%"
            return progressBar
        }
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
                script {
                    def progressBar = generateProgressBar(1)
                    slackSend(
                        tokenCredentialId: 'SlackToken',
                        channel: params.SLACK_CHANNEL,
                        color: 'good',
                        message: """
                            :white_check_mark: *Checkout Complete*
                            ${progressBar}
                            *Stage*: 1/${TOTAL_STAGES} - Source code checkout completed
                        """
                    )
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def dockerImage = "${DOCKER_IMAGE}"
                    def envTag = "${params.environment}"
                    
                    // Send start notification to Slack
                    try {
                        def startMessage = """
                            :construction: *Build Started* :gear:
                            *Job*: ${env.JOB_NAME}
                            *Build Number*: #${env.BUILD_NUMBER}
                            *Branch*: :git: ${params.branch}
                            *Environment*: :gear: ${params.environment}
                            *Image*: :whale: ${dockerImage}:${envTag}
                            
                            :arrow_forward: Starting Docker build...
                            • <${env.BUILD_URL}|View Build Progress>
                        """
                        
                        slackSend(
                            tokenCredentialId: 'SlackToken',
                            channel: params.SLACK_CHANNEL,
                            color: 'warning',
                            message: startMessage
                        )
                    } catch (Exception e) {
                        echo "Failed to send start notification to Slack: ${e.message}"
                    }
                    
                    // Print variables for debugging
                    echo "Debug info:"
                    echo "dockerImage = ${dockerImage}"
                    echo "envTag = ${envTag}"
                    echo "Full image name will be: ${dockerImage}:${envTag}"
                    
                    // Build the image
                    echo "Executing build command: docker build -t ${dockerImage}:${envTag} ."
                    sh "docker build -t ${dockerImage}:${envTag} ."
                    
                    // Tag as latest
                    echo "Executing tag command: docker tag ${dockerImage}:${envTag} ${dockerImage}:latest"
                    sh "docker tag ${dockerImage}:${envTag} ${dockerImage}:latest"
                    
                    // List images
                    sh "docker images | grep ${dockerImage} || true"
                    def progressBar = generateProgressBar(2)
                    slackSend(
                        tokenCredentialId: 'SlackToken',
                        channel: params.SLACK_CHANNEL,
                        color: 'good',
                        message: """
                            :white_check_mark: *Docker Build Complete*
                            ${progressBar}
                            *Stage*: 2/${TOTAL_STAGES} - Docker image built successfully
                        """
                    )
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // Send deployment start notification
                    try {
                        def deployStartMessage = """
                            :rocket: *Deployment Started* :gear:
                            *Job*: ${env.JOB_NAME}
                            *Build Number*: #${env.BUILD_NUMBER}
                            *Branch*: :git: ${params.branch}
                            *Environment*: :gear: ${params.environment}
                            *Image*: :whale: ${DOCKER_IMAGE}:latest
                            *Target Port*: :computer: ${PORT}
                            
                            :arrow_forward: Starting deployment process...
                            • Container will be recreated
                            • Application will be available at port ${PORT}
                            • <${env.BUILD_URL}console|View Deployment Progress>
                        """
                        
                        slackSend(
                            tokenCredentialId: 'SlackToken',
                            channel: params.SLACK_CHANNEL,
                            color: 'warning',
                            message: deployStartMessage
                        )
                    } catch (Exception e) {
                        echo "Failed to send deployment start notification to Slack: ${e.message}"
                    }
                }
                
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
                def progressBar = generateProgressBar(3)
                slackSend(
                    tokenCredentialId: 'SlackToken',
                    channel: params.SLACK_CHANNEL,
                    color: 'good',
                    message: """
                        :white_check_mark: *Deployment Complete*
                        ${progressBar}
                        *Stage*: 3/${TOTAL_STAGES} - Application deployed successfully
                    """
                )
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
                    :clipboard: *Build Summary* :mag:
                    *Build Status*: ${buildStatus == 'SUCCESS' ? ':white_check_mark:' : ':x:'} ${buildStatus}
                    *Job*: ${env.JOB_NAME}
                    *Build Number*: #${env.BUILD_NUMBER}
                    *Branch*: :git: ${params.branch}
                    *Environment*: :gear: ${params.environment}
                    *Duration*: :hourglass_flowing_sand: ${buildDuration}
                    *Build URL*: :link: ${env.BUILD_URL}
                    *Image Tag*: :whale: ${DOCKER_IMAGE}:${params.environment}
                    *Deployed Port*: :computer: ${PORT}
                    *App URL*: :link: <http://your-server:${PORT}|Open Application>

                    ${buildStatus == 'SUCCESS' 
                        ? ':rocket: Application deployed successfully!' 
                        : ':boom: Build failed - Check logs for details'}
                    
                    :memo: *View Details*
                    • <${env.BUILD_URL}console|View Build Logs>
                    • <${env.BUILD_URL}|View Build Page>
                    • <${env.JOB_URL}|View Project>
                """
                
                echo "Sending notification to Slack channel: ${params.SLACK_CHANNEL}"
                try {
                    slackSend(
                        tokenCredentialId: 'SlackToken',
                        channel: params.SLACK_CHANNEL,
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
