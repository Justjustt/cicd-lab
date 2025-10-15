pipeline {
    agent any
    
    tools {
        nodejs 'Node 7.8.0'
    }
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_MAIN = 'nodemain:v1.0'
        DOCKER_IMAGE_DEV = 'nodedev:v1.0'
        PORT_MAIN = '3000'
        PORT_DEV = '3001'
    }
    
    stages {
        stage('Checkout SCM') {
            steps {
                echo "Checking out from branch: ${BRANCH_NAME ?: env.GIT_BRANCH}"
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
                    if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == null) {
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
                    
                    if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == null) {
                        dockerImage = "${DOCKER_IMAGE_MAIN}"
                        port = "${PORT_MAIN}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        dockerImage = "${DOCKER_IMAGE_DEV}"
                        port = "${PORT_DEV}"
                    }
                    
                    echo "Deploying ${dockerImage} on port ${port}..."
                    
                    // Stop and remove existing containers
                    sh '''
                        docker ps -a | grep -E "${DOCKER_IMAGE_MAIN}|${DOCKER_IMAGE_DEV}" | awk '{print $1}' | xargs -r docker stop
                        docker ps -a | grep -E "${DOCKER_IMAGE_MAIN}|${DOCKER_IMAGE_DEV}" | awk '{print $1}' | xargs -r docker rm
                    '''
                    
                    // Run new container based on branch
                    if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == null) {
                        sh "docker run -d --expose ${PORT_MAIN} -p ${PORT_MAIN}:3000 ${dockerImage}"
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh "docker run -d --expose ${PORT_DEV} -p ${PORT_DEV}:3000 ${dockerImage}"
                    }
                    
                    echo "Deployment complete! Access application at http://localhost:${port}"
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
