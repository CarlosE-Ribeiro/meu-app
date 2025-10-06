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
        NVD_DELAY_MS = '30000'           // Delay para contornar rate limit
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

        stage('Dependency Check') {
  steps {
    bat """
      if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
      set FAILCVSS=%FAIL_CVSS%
      if "%%FAILCVSS%%"=="" set FAILCVSS=0

      mvn -B org.owasp:dependency-check-maven:check ^
        -Dformat=HTML,XML ^
        -Ddata="%DC_CACHE%" ^
        -Dnoupdate ^
        -DfailOnCVSS=%%FAILCVSS%% ^
        -DfailOnError=false
    """
  }
  post {
    always {
      // Publica só se o XML existir (evita UNSTABLE por ausência de arquivo)
      script {
        def xml = 'target/dependency-check-report.xml'
        def html = 'target/dependency-check-report.html'
        if (fileExists(xml)) {
          dependencyCheckPublisher pattern: xml
        } else {
          echo "Dependency-Check: relatório XML não encontrado (provável sem análise)."
        }
        if (fileExists(html)) {
          archiveArtifacts artifacts: html, allowEmptyArchive: true
        } else {
          echo "Dependency-Check: relatório HTML não encontrado."
        }
      }
    }
  }
}

    }
        
    post {
        always {
            echo "Pipeline finalizado"
        }
    }
}
