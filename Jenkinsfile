 pipeline {
    agent any
  
    stages {
        stage('Clone sources') {
            steps {
                git branch: 'main', url: 'https://github.com/globalhitss-devops/juice-shop.git'
            }
        }
    
    }
    post {
        always {
            echo 'Removing container'
            sh '''
                     docker stop owasp
                     docker rm owasp
                 '''
            cleanWs()
        }
    }
 }
