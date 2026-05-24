pipeline {
    agent any

    environment {
        // ==========================================
        // 1. DOCKER HUB CONFIGURATION
        // ==========================================
        // The ID of the username/password credential stored in Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'
        // Your Docker Hub username
        DOCKERHUB_USERNAME       = 'pranayyy123'
        // The name of the Docker repository
        IMAGE_NAME               = 'bike-backend'
        // Create a unique image tag using the build number
        IMAGE_TAG                = "${env.BUILD_NUMBER}"

        // ==========================================
        // 2. TARGET DEPLOYMENT EC2 CONFIGURATION
        // ==========================================
        // The ID of the SSH Username with Private Key credential stored in Jenkins
        DEPLOY_SSH_CREDENTIALS_ID = 'ec2-ssh-key'
        // Public IP or DNS of the remote EC2 instance where Docker is installed
        DEPLOY_HOST               = '172.31.31.240'

        // ==========================================
        // 3. SECURE CONFIGURATION (CREDENTIALS)
        // ==========================================
        // Jenkins Secret Text credential ID for the Database URL
        DB_URL_CREDENTIAL_ID     = 'bike-backend-db-url'
        // Jenkins Secret Text credential ID for the application port
        PORT_CREDENTIAL_ID       = 'bike-backend-port'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image...'
                sh "docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Logging in to Docker Hub and pushing image...'
                withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Target EC2') {
            steps {
                echo 'Deploying to remote EC2 instance via SSH...'
                // Retrieve the SSH private key, DB URL, and Port securely from Jenkins credentials
                withCredentials([
                    sshUserPrivateKey(credentialsId: env.DEPLOY_SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    string(credentialsId: env.DB_URL_CREDENTIAL_ID, variable: 'DATABASE_URL'),
                    string(credentialsId: env.PORT_CREDENTIAL_ID, variable: 'APP_PORT')
                ]) {
                    sh """
                        ssh -i \${SSH_KEY} -o StrictHostKeyChecking=no \${SSH_USER}@\${DEPLOY_HOST} "
                            echo 'Connected to remote EC2. Updating application...'
                            
                            # Pull the latest image
                            docker pull \${DOCKERHUB_USERNAME}/\${IMAGE_NAME}:latest
                            
                            # Stop and remove the existing container if it is running
                            docker stop \${IMAGE_NAME} || true
                            docker rm \${IMAGE_NAME} || true
                            
                            # Run the new container using the retrieved secrets
                            docker run -d \\
                              --name \${IMAGE_NAME} \\
                              --network bike-network \\
                              -e PORT=\${APP_PORT} \\
                              -e DATABASE_URL='\${DATABASE_URL}' \\
                              \${DOCKERHUB_USERNAME}/\${IMAGE_NAME}:latest
                              
                            echo 'Deployment completed successfully!'
                        "
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up local Docker images on the Jenkins agent...'
            sh "docker rmi \${DOCKERHUB_USERNAME}/\${IMAGE_NAME}:\${IMAGE_TAG} || true"
            sh "docker rmi \${DOCKERHUB_USERNAME}/\${IMAGE_NAME}:latest || true"
        }
    }
}
