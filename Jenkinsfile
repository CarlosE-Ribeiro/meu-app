pipeline {
    agent any

    tools {
        jdk 'jdk-21'           // Exemplo: JDK 21 instalado em "Manage Jenkins » Tools"
        maven 'maven-3.9'      // Exemplo: Maven 3.x instalado em "Manage Jenkins » Tools"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    //parameters {
    //    booleanParam(name: 'RUN_DEP_SCAN', defaultValue: true, description: 'Executar OWASP Dependency-Check?')
    //    string(name: 'FAIL_CVSS', defaultValue: '7.0', description: 'Falhar build se CVSS >= (ex.: 7.0). Use 0 para nunca falhar.')
    //}

    environment {
        DC_CACHE = 'C:\\DC_CACHE'        // Pasta de cache local do Dependency-Check
        NVD_DELAY_MS = '300000'           // Delay para contornar rate limit
        NVD_RETRIES = '15'
        NVD_CF_RETRIES = '15'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat '''
                    mvn -B -DskipTests clean package
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
                    fingerprint 'target/*.jar'
                }
            }
        }

        stage('Test') {
            steps {
                bat '''
                    mvn -B test
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan target/', odcInstallation: 'OWASP-DC'
            }
        }

        stage('Publish OWASP Dependency Check Report') {
            steps {
                publishHTML(target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'target',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
            }
        }

    }
        
    post {
        always {
            echo "Pipeline finalizado"
        }
    }
}
