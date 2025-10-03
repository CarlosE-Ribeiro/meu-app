pipeline {
  agent any

  // Ajuste os nomes conforme "Manage Jenkins » Tools"
  tools {
    jdk   'jdk-21'     // ex.: JDK 21 instalado no Jenkins
    maven 'maven-3.9'    // ex.: Maven 3.x instalado no Jenkins
  }

   options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    booleanParam(name: 'RUN_DEP_SCAN', defaultValue: true, description: 'Executar OWASP Dependency-Check?')
    string(name: 'FAIL_CVSS', defaultValue: '7.0', description: 'Falhar build se CVSS >= (ex.: 7.0). Use 0 para nunca falhar.')
  }

  environment {
    // Cache persistente do Dependency-Check (crie a pasta uma vez)
    DC_CACHE = 'C:\\DC_CACHE'

    // Se tiver salvo sua chave da NVD em "Credentials → Secret text" com ID NVD_API_KEY,
    // descomente a linha abaixo e comente a de cima:
    // NVD_API_KEY = credentials('NVD_API_KEY')
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
        success {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('Test') {
      steps {
        bat 'mvn -B test'
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('OWASP Dependency-Check') {
    options { timeout(time: 5, unit: 'MINUTES') }
    steps {
        bat 'if not exist "C:\\DC_CACHE" mkdir "C:\\DC_CACHE"'
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
        bat '''
            mvn -B -DskipTests org.owasp:dependency-check-maven:check ^
                -Dnvd.api.key=%NVD_API_KEY% ^
                -Dnvd.api.delay=15000 ^
                -Dnvd.api.cloudflare.retries=6 ^
                -DdataDirectory=C:\\DC_CACHE ^
                -Dautoupdate=false
                -Ddependency-check.failOnError=false
                -Ddependency-check.quickQueryTimestamp=true ^
                -Ddependency-check.cve.validForHours=24 ^
                -Danalyzers.pg.enabled=false ^
                -Danalyzer.node.audit.enabled=false ^
                -Danalyzer.nuspec.enabled=false ^
                -Danalyzer.assembly.enabled=false
        '''
        }
    }
    post {
        always {
        // publique os relatórios se quiser vê-los no Jenkins
        dependencyCheckPublisher()
        archiveArtifacts artifacts: 'target/dependency-check-report.*', allowEmptyArchive: true
        }
    }
    }


  }

  post {
    always {
      echo "Build: ${currentBuild.currentResult}"
    }
  }
}