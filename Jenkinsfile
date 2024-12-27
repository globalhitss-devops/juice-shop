pipeline {
    agent any
    
    tools {nodejs "NodeJS-18-20-4"}

    environment {
        SONAR_TOKEN = credentials('global-sonar-token')
        DOJO_TOKEN = credentials('global-dojo-token')
    }

    stages {
        stage('Clone sources') {
            steps {
                git branch: 'main', credentialsId: 'github-global', url: 'https://github.com/globalhitss-devops/juice-shop.git'
            }
        }



        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }        
        
    }
}
