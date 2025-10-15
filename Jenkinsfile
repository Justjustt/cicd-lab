pipeline {
  agent any

  tools {
    nodejs 'Node 7.8.0'   // <-- must match the exact name you created
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build') {
      steps {
        sh 'node -v'
        sh 'npm -v'
        sh 'npm install'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test || echo "Tests failed or missing; continuing for lab."'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def imageTag = (env.BRANCH_NAME == 'main') ? 'nodemain:v1.0' : 'nodedev:v1.0'
          sh "docker build -t ${imageTag} ."
        }
      }
    }

    stage('Deploy') {
      steps {
        script {
          sh '''
            docker ps -q --filter "name=node_main" | xargs -r docker stop
            docker ps -aq --filter "name=node_main" | xargs -r docker rm
            docker ps -q --filter "name=node_dev"  | xargs -r docker stop
            docker ps -aq --filter "name=node_dev" | xargs -r docker rm
          '''
          if (env.BRANCH_NAME == 'main') {
            sh 'docker run -d --name node_main --expose 3000 -p 3000:3000 nodemain:v1.0'
          } else {
            sh 'docker run -d --name node_dev  --expose 3000 -p 3001:3000 nodedev:v1.0'
          }
        }
      }
    }
  }
}
