pipeline {
    agent any

    tools {
        jdk   'jdk-21'
        maven 'maven-3.9'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    parameters {
        booleanParam(name: 'RUN_DEP_SCAN', defaultValue: true, description: 'Executar OWASP Dependency-Check?')
        string(name: 'FAIL_CVSS', defaultValue: '7.0', description: 'Falhar build se CVSS >= (ex.: 7.0). Use 11 para nunca falhar.')
    }

    environment {
        // Cache dos dados do NVD dentro do workspace do job
        DC_CACHE = "${env.WORKSPACE}@tmp\\DC_CACHE"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat 'mvn -B -DskipTests clean package'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
                }
            }
        }

        stage('Test') {
            steps {
                bat 'mvn -B test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
                }
            }
        }
        
        stage('OWASP Dependency-Check') {
            when {
                expression { params.RUN_DEP_SCAN }
            }
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    script {
                        // Esta linha agora funcionará, pois o Jenkins sabe como instalar 'OWASP-DC'
                        def dcPath = tool name: 'OWASP-DC', type: 'dependencyCheck'

                        // Comando corrigido para Windows
                        bat """
                            call "${dcPath}\\bin\\dependency-check.bat" --scan "." --format "ALL" --prettyPrint --apiKey "%NVD_API_KEY%" --project "Meu Projeto" --failOnCVSS ${params.FAIL_CVSS} --data "%DC_CACHE%"
                        """
                    }
                }
                
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
    }
    post {
        always {
            echo "Pipeline finalizado"
        }
    }
}