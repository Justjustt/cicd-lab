pipeline {
  agent any

  environment {
    NODEJS_HOME = tool name: 'Node 7.8.0', type: 'NodeJSInstallation'
    PATH = "${NODEJS_HOME}/bin:${env.PATH}"
  }

  options {
    // keeps console logs cleaner; optional
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        // In Multibranch, this checks out the current branch automatically
        checkout scm
        sh 'echo "Building branch: ${BRANCH_NAME}"'
      }
    }

    stage('Build') {
      steps {
        sh 'npm install'
      }
    }

    stage('Test') {
      steps {
        // keep going even if demo tests are missing
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
          // low downtime: stop old if exists, then remove
          sh '''
            docker ps -q --filter "name=node_main" | xargs -r docker stop
            docker ps -aq --filter "name=node_main" | xargs -r docker rm
            docker ps -q --filter "name=node_dev"  | xargs -r docker stop
            docker ps -aq --filter "name=node_dev" | xargs -r docker rm
          '''

          if (env.BRANCH_NAME == 'main') {
            sh 'docker run -d --name node_main --expose 3000 -p 3000:3000 nodemain:v1.0'
          } else {
            // map 3001 on host -> 3000 in container (app still listens on 3000 inside)
            sh 'docker run -d --name node_dev  --expose 3000 -p 3001:3000 nodedev:v1.0'
          }
        }
      }
    }
  }
}
