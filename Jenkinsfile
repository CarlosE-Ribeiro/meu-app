pipeline {
  agent any

  tools {
    jdk 'jdk-21'
    maven 'maven-3.9'
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
    DC_CACHE = 'C:\\DC_CACHE'
    NVD_DELAY_MS = '30000'
    NVD_RETRIES = '15'
    NVD_CF_RETRIES = '15'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build') {
      steps {
        bat 'mvn -B -DskipTests clean package'
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
        bat 'mvn -B test'
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
      }
    }

    stage('Dependency Check') {
      when { expression { return params.RUN_DEP_SCAN } }
      steps {
        // garante o cache no Windows
        bat '''
          if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
        '''
        // roda o scanner (gera HTML e XML no workspace)
        dependencyCheck(
          odcInstallation: 'OWASP-DC',                // <- deve bater com o nome no Global Tool
          additionalArguments: "--data \"%DC_CACHE%\" --format HTML --format XML --nvdApiDelay %NVD_DELAY_MS% --nvdMaxRetryCount %NVD_RETRIES% --nvdApiMaxRetryCount %NVD_CF_RETRIES% --failOnCVSS ${FAIL_CVSS}"
        )
      }
      post {
        always {
          // publica o relatório (o plugin espera esse nome por padrão)
          dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
          // opcional: arquiva também o HTML para consulta
          archiveArtifacts artifacts: 'dependency-check-report.html', allowEmptyArchive: true
        }
      }
    }
  }

  post {
    always { echo 'Pipeline finalizado' }
  }
}
