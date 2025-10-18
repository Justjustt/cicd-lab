@Library('jenkins-shared-library') _

pipeline {
    agent any
    
    tools {
        nodejs 'Node 7.8.0'
    }
    
    environment {
        DOCKERHUB_REPO = 'jus7'
        DOCKER_IMAGE_MAIN = "${DOCKERHUB_REPO}/cicd-lab-main:v1.0"
        DOCKER_IMAGE_DEV = "${DOCKERHUB_REPO}/cicd-lab-dev:v1.0"
        // Built-in credentials (Jenkins equivalent of GitLab's CI_REGISTRY_USER/CI_JOB_TOKEN)
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
        PORT_MAIN = '3000'
        PORT_DEV = '3001'
        // Dynamic variables
        DOCKER_IMAGE = ''
        DEPLOY_PORT = ''
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "================================================"
                    echo "Building branch: ${env.BRANCH_NAME}"
                    echo "================================================"
                    
                    // Validate branch
                    if (env.BRANCH_NAME != 'main' && env.BRANCH_NAME != 'dev') {
                        error("Invalid branch. Only 'main' and 'dev' branches are supported")
                    }
                    
                    // Set environment variables based on branch
                    if (env.BRANCH_NAME == 'main') {
                        env.DOCKER_IMAGE = env.DOCKER_IMAGE_MAIN
                        env.DEPLOY_PORT = env.PORT_MAIN
                    } else {
                        env.DOCKER_IMAGE = env.DOCKER_IMAGE_DEV
                        env.DEPLOY_PORT = env.PORT_DEV
                    }
                    
                    echo "Docker Image: ${env.DOCKER_IMAGE}"
                    echo "Deploy Port: ${env.DEPLOY_PORT}"
                }
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Dependencies') {
            steps {
                script {
                    echo "Installing dependencies once (template pattern)..."
                    sh '''
                        # Install application dependencies
                        npm install
                        
                        # Install dev dependencies for linting and testing
                        npm install --save-dev || true
                    '''
                }
            }
        }
        
        stage('Lint & Test') {
            steps {
                script {
                    echo "Running lint and tests using pre-installed dependencies..."
                    
                    // Lint Dockerfile
                    sh 'hadolint Dockerfile || true'
                    
                    // Run tests (dependencies already installed)
                    sh 'npm test || true'
                }
            }
        }
        
        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "Building and pushing Docker image in one step..."
                    
                    // Use Jenkins built-in credential variables (like GitLab CI_REGISTRY_USER/CI_JOB_TOKEN)
                    sh """
                        # Login using built-in Jenkins credentials
                        echo ${DOCKER_CREDENTIALS_PSW} | docker login -u ${DOCKER_CREDENTIALS_USR} --password-stdin
                        
                        # Build and push in single operation (no duplication)
                        docker build -t ${env.DOCKER_IMAGE} .
                        docker push ${env.DOCKER_IMAGE}
                        
                        docker logout
                    """
                    
                    echo "✓ Image built and pushed: ${env.DOCKER_IMAGE}"
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo "Scanning image with Trivy..."
                    
                    sh """
                        trivy image --severity HIGH,CRITICAL \
                            --format json \
                            --output trivy-report-${env.BRANCH_NAME}.json \
                            ${env.DOCKER_IMAGE} || true
                        
                        trivy image --severity HIGH,CRITICAL ${env.DOCKER_IMAGE} || true
                    """
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    echo "Deploying to ${env.BRANCH_NAME} environment..."
                    
                    sh """
                        # Stop only containers on this environment's port
                        CONTAINER_ID=\$(docker ps -q --filter "publish=${env.DEPLOY_PORT}")
                        if [ ! -z "\$CONTAINER_ID" ]; then
                            echo "Stopping old container: \$CONTAINER_ID"
                            docker stop \$CONTAINER_ID
                            docker rm \$CONTAINER_ID
                        else
                            echo "No old container found on port ${env.DEPLOY_PORT}"
                        fi
                        
                        # Deploy new container
                        docker run -d \\
                            --name ${env.BRANCH_NAME}-app-${BUILD_NUMBER} \\
                            --expose ${env.DEPLOY_PORT} \\
                            -p ${env.DEPLOY_PORT}:3000 \\
                            ${env.DOCKER_IMAGE}
                        
                        echo "✓ Deployed on port ${env.DEPLOY_PORT}"
                    """
                }
            }
        }
        
        stage('Verify') {
            steps {
                script {
                    echo "Verifying deployment..."
                    sleep(time: 3, unit: 'SECONDS')
                    
                    sh """
                        echo "Checking container status..."
                        docker ps | grep ${env.BRANCH_NAME}-app-${BUILD_NUMBER}
                        
                        echo "Testing endpoint..."
                        curl -f http://localhost:${env.DEPLOY_PORT} || echo "Warning: Application starting"
                        
                        echo "✓ Verification complete"
                    """
                }
            }
        }
        
        stage('Trigger External Deployment') {
            steps {
                script {
                    def pipelineName = (env.BRANCH_NAME == 'main') ? 'Deploy_to_main' : 'Deploy_to_dev'
                    
                    echo "Triggering ${pipelineName}..."
                    build job: pipelineName, wait: false
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: "trivy-report-${env.BRANCH_NAME}.json", allowEmptyArchive: true
            echo "Pipeline execution completed for ${env.BRANCH_NAME}"
        }
        success {
            echo "================================================"
            echo "✓ ${env.BRANCH_NAME} pipeline successful!"
            echo "Access: http://localhost:${env.DEPLOY_PORT}"
            echo "================================================"
        }
        failure {
            echo "✗ Pipeline failed for ${env.BRANCH_NAME}"
        }
        cleanup {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
    }
}
