@Library('jenkins-shared-library') _

pipeline {
    agent any
    
    tools {
        nodejs 'Node 7.8.0'
    }
    
    environment {
        DOCKERHUB_REPO = 'your-dockerhub-username'
        DOCKER_IMAGE_MAIN = "${DOCKERHUB_REPO}/cicd-lab-main:v1.0"
        DOCKER_IMAGE_DEV = "${DOCKERHUB_REPO}/cicd-lab-dev:v1.0"
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
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
        
        stage('Lint Dockerfile') {
            steps {
                echo "Linting Dockerfile with Hadolint..."
                sh 'hadolint Dockerfile || true'
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
        
        stage('Scan with Trivy') {
            steps {
                script {
                    def dockerImage
                    if (env.BRANCH_NAME == 'main') {
                        dockerImage = "${DOCKER_IMAGE_MAIN}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        dockerImage = "${DOCKER_IMAGE_DEV}"
                    }
                    
                    echo "Scanning image with Trivy: ${dockerImage}"
                    
                    sh """
                        trivy image --severity HIGH,CRITICAL \\
                            --format json \\
                            --output trivy-report.json \\
                            ${dockerImage} || true
                        
                        trivy image --severity HIGH,CRITICAL \\
                            ${dockerImage} || true
                    """
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    def dockerImage
                    if (env.BRANCH_NAME == 'main') {
                        dockerImage = "${DOCKER_IMAGE_MAIN}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        dockerImage = "${DOCKER_IMAGE_DEV}"
                    }
                    
                    echo "Pushing image to Docker Hub: ${dockerImage}"
                    
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${dockerImage}
                            docker logout
                        """
                    }
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
                    
                    sh """
                        CONTAINER_ID=\$(docker ps -q --filter "publish=${port}")
                        if [ ! -z "\$CONTAINER_ID" ]; then
                            docker stop \$CONTAINER_ID
                            docker rm \$CONTAINER_ID
                        fi
                        
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
        
        stage('Trigger Deployment Pipeline') {
            steps {
                script {
                    def pipelineName
                    if (env.BRANCH_NAME == 'main') {
                        pipelineName = 'Deploy_to_main'
                    } else if (env.BRANCH_NAME == 'dev') {
                        pipelineName = 'Deploy_to_dev'
                    }
                    
                    echo "Triggering ${pipelineName}..."
                    build job: pipelineName, wait: false
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
            echo "Pipeline execution completed"
        }
        success {
            echo "✓ Build and deployment successful!"
        }
        failure {
            echo "✗ Build or deployment failed!"
        }
    }
}
