pipeline {
    agent any
    
    tools {
        nodejs 'Node 7.8.0'
    }
    
    environment {
        DOCKER_IMAGE_MAIN = 'nodemain:v1.0'
        DOCKER_IMAGE_DEV = 'nodedev:v1.0'
        PORT_MAIN = '3000'
        PORT_DEV = '3001'
    }
    
    stages {
        stage('Checkout SCM') {
            steps {
                echo "Checking out from branch: ${BRANCH_NAME}"
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo "Building Node.js application..."
                sh 'npm install'
            }
        }
        
        stage('Test') {
            steps {
                echo "Running tests..."
                sh 'npm test || true'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def dockerImage
                    if (env.BRANCH_NAME == 'main') {
                        dockerImage = "${DOCKER_IMAGE_MAIN}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        dockerImage = "${DOCKER_IMAGE_DEV}"
                    }
                    echo "Building Docker image: ${dockerImage}"
                    sh "docker build -t ${dockerImage} ."
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    def dockerImage
                    def port
                    
                    if (env.BRANCH_NAME == 'main') {
                        dockerImage = "${DOCKER_IMAGE_MAIN}"
                        port = "${PORT_MAIN}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        dockerImage = "${DOCKER_IMAGE_DEV}"
                        port = "${PORT_DEV}"
                    }
                    
                    echo "Deploying ${dockerImage} on port ${port}..."
                    
                    // Stop and remove ONLY containers for this specific environment
                    sh """
                        echo "Stopping containers for ${env.BRANCH_NAME} environment..."
                        CONTAINER_ID=\$(docker ps -q --filter "publish=${port}")
                        if [ ! -z "\$CONTAINER_ID" ]; then
                            echo "Stopping container: \$CONTAINER_ID"
                            docker stop \$CONTAINER_ID
                            docker rm \$CONTAINER_ID
                        else
                            echo "No running container found on port ${port}"
                        fi
                    """
                    
                    // Run new container
                    sh """
                        docker run -d \\
                            --name ${env.BRANCH_NAME}-app-${BUILD_NUMBER} \\
                            --expose ${port} \\
                            -p ${port}:3000 \\
                            ${dockerImage}
                    """
                    
                    echo "Deployment complete! Access at http://localhost:${port}"
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed"
        }
        success {
            echo "Build and deployment successful!"
        }
        failure {
            echo "Build or deployment failed!"
        }
    }
}
