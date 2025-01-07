def scan_type
def target
 pipeline {
    agent any

    tools {nodejs "NodeJS-18-20-4"}
  
    environment {
        SONAR_TOKEN = credentials('global-sonar-token')
        DOJO_TOKEN = credentials('global-dojo-token')
    }

    parameters {
        choice  choices: ['Baseline', 'APIS', 'Full'],
                 description: 'Type of scan that is going to perform inside the container',
                 name: 'SCAN_TYPE'

        string defaultValue: 'https://juice-shop.herokuapp.com',
                 description: 'Target URL to scan',
                 name: 'TARGET'

        booleanParam defaultValue: true,
                 description: 'Parameter to know if wanna generate report.',
                 name: 'GENERATE_REPORT'
    }
    stages {
        stage('Clone sources') {
            steps {
                git branch: 'master', credentialsId: 'github-global', url: 'https://github.com/globalhitss-devops/juice-shop.git'
            }
        }

        stage('Clone scripts CI/CD') {
            steps {
                dir("devops") {
                     git branch: 'main', credentialsId: 'github-global', url: 'https://github.com/globalhitss-devops/scripts-cicd.git'
                 }
            }
        }

        stage("SonarQube Analysis"){
            environment {
                SCANNER_HOME = tool 'SonarQubeScanner';    
            }

            steps{
               withSonarQubeEnv("SonarQube"){
                   sh '''$SCANNER_HOME/bin/sonar-scanner \
                   -Dsonar.projectName=juice-shop \
                   -Dsonar.projectKey=juice-shop \
                   -Dsonar.sources=. \
                   -Dsonar.language=ts \
                   -Dsonar.sourceEncoding=UTF-8 \
                   -Dsonar.scm.disabled=true \
                   -Dsonar.issuesReport.html.enable=true \
                   -Dsonar.report.export.path=report.json'''
               }
            }
        }

        stage('SonarQube Report Export') {
            steps {
                script {
                    def sonarServer = 'http://52.200.180.65:9000'
                    def sonarReportData = sh (
                    script: "curl -s -u ${SONAR_TOKEN}: ${sonarServer}/api/issues/search?componentKeys=juice-shop",
                    returnStdout: true
                    ).trim()
                    writeFile file: 'sonarqube-report.json', text: sonarReportData
                }
            }
        }

        stage ("Import Sonarqube Defect DOJO") {
            steps {
                script {
                    sh '''
                        cp ${WORKSPACE}/sonarqube-report.json ${WORKSPACE}/devops/defect-dojo/sonarqube-report.json
                     ''' 
                    dir("${env.WORKSPACE}/devops/defect-dojo"){
                        sh "python3 upload-reports-sonarqube.py sonarqube-report.json"
                        
                        sh """python3 --version /
                        pwd /
                        ls -lha"""
                    }
                }
            }
        }

        stage("SonarQube Quality Gates"){
            steps{
               timeout(time: 1, unit: "MINUTES"){
                   waitForQualityGate abortPipeline: false
               }
            }
        }
        
//        stage("OWASP Dependency-Check"){
//            steps{
//                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'OWASP'
//                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
//            }
//        }

        stage('OWASP Dependency-Check') {
              steps {
                dependencyCheck additionalArguments: ''' 
                            -o './'
                            -s './'
                            -f 'ALL' 
                            --prettyPrint''', odcInstallation: 'OWASP'
                
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        } 

        stage ("Import OWASP Defect DOJO") {
            steps {
                script {
                    sh '''
                        cp ${WORKSPACE}/dependency-check-report.xml ${WORKSPACE}/devops/defect-dojo/dependency-check-report.xml
                     ''' 
                    dir("${env.WORKSPACE}/devops/defect-dojo"){
                        sh "python3 upload-reports-dependency-check.py dependency-check-report.xml"
                        
                        sh """python3 --version /
                        pwd /
                        ls -lha"""
                    }
                }
            }
        }

        stage("TRIVY Scan") {            
            steps {                
                sh "trivy fs -f json . > trivy-fs_report.json"
            }        
        }

        stage ("Import TRIVY Defect DOJO") {
            steps {
                script {
                    sh '''
                        cp ${WORKSPACE}/trivy-fs_report.json ${WORKSPACE}/devops/defect-dojo/trivy-fs_report.json
                     ''' 
                    dir("${env.WORKSPACE}/devops/defect-dojo"){
                        sh "python3 upload-reports-trivy.py trivy-fs_report.json"
                        
                        sh """python3 --version /
                        pwd /
                        ls -lha"""
                    }
                }
            }
        }
     
        stage('Pipeline Info') {
            steps {
                script {
                    echo '<--Parameter Initialization-->'
                    echo """
                         The current parameters are:
                             Scan Type: ${params.SCAN_TYPE}
                             Target: ${params.TARGET}
                             Generate report: ${params.GENERATE_REPORT}
                         """
                }
            }
        }

        stage('Setting up OWASP ZAP docker container') {
            steps {
                echo 'Pulling up last OWASP ZAP container --> Start'
                sh 'docker pull ghcr.io/zaproxy/zaproxy:stable'
                echo 'Pulling up last VMS container --> End'
                echo 'Starting container --> Start'
                sh 'docker run -dt --name owasp ghcr.io/zaproxy/zaproxy:stable /bin/bash '
            }
        }

        stage('Prepare wrk directory') {
            when {
                environment name : 'GENERATE_REPORT', value: 'true'
            }
            steps {
                script {
                    sh '''
                             docker exec owasp \
                             mkdir /zap/wrk
                         '''
                }
            }
        }

        stage('Scanning target on owasp container') {
            steps {
                script {
                    scan_type = "${params.SCAN_TYPE}"
                    echo "----> scan_type: $scan_type"
                    target = "${params.TARGET}"
                    if (scan_type == 'Baseline') {
                        sh """
                             docker exec owasp \
                             zap-baseline.py \
                             -t $target \
                             -g gen.conf \
                             -x report.xml \
                             -I
                         """
                    }
                     else if (scan_type == 'APIS') {
                        sh """
                             docker exec owasp \
                             zap-api-scan.py \
                             -t $target \
                             -x report.xml \
                             -f openapi \
                             -I
                         """
                     }
                     else if (scan_type == 'Full') {
                        sh """
                             docker exec owasp \
                             zap-full-scan.py \
                             -t $target \
                             -g gen.conf \
                             -x report.xml \
                             -I
                         """
                     }
                     else {
                        echo 'Something went wrong...'
                     }
                }
            }
        }

        stage('Copy Report to Workspace') {
            steps {
                script {
                    sh '''
                         docker cp owasp:/zap/wrk/report.xml ${WORKSPACE}/report.xml
                     '''
                }
            }
        }

        stage ("Import OWASP ZAP Defect DOJO") {
            steps {
                script {
                    sh '''
                        cp ${WORKSPACE}/report.xml ${WORKSPACE}/devops/defect-dojo/report.xml
                     ''' 
                    dir("${env.WORKSPACE}/devops/defect-dojo"){
                        sh "python3 upload-reports-owasp-zap.py report.xml"
                        
                        sh """python3 --version /
                        pwd /
                        ls -lha"""
                    }
                }
            }
        }
     
        stage('Email Report'){
            steps{
                emailext (
                attachLog: true,
                // attachmentsPattern: '**/*.html',
                attachmentsPattern: '**/report.xml',
                body: "Please find the attached report for the latest OWASP ZAP Scan.",
                recipientProviders: [buildUser()],
                subject: "OWASP ZAP Report",
                to: 'claudio.bianco.3@globalhitss.com.br'
                )
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
