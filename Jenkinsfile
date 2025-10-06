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
    withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
      bat '''
        if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"

        rem 1) Popula o cache local (primeira vez ou quando quiser atualizar a base)
        mvn -B org.owasp:dependency-check-maven:update-only ^
          -Ddata="%DC_CACHE%" ^
          -DnvdApiKey=%NVD_API_KEY% ^
          -DnvdApiDelay=%NVD_DELAY_MS% ^
          -DnvdMaxRetryCount=%NVD_RETRIES%

        rem 2) Executa o scan usando o cache; se quiser nunca falhar por erro interno, acrescente -DfailOnError=false
        mvn -B org.owasp:dependency-check-maven:check ^
          -Dformat=HTML,XML ^
          -Ddata="%DC_CACHE%" ^
          -DnvdApiKey=%NVD_API_KEY% ^
          -DnvdApiDelay=%NVD_DELAY_MS% ^
          -DnvdMaxRetryCount=%NVD_RETRIES% ^
          -DfailOnCVSS=%FAIL_CVSS%
      '''
    }
  }
  post {
    always {
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      archiveArtifacts artifacts: 'target/dependency-check-report.html', allowEmptyArchive: true
    }
  }
}

  }

  post {
    always { echo 'Pipeline finalizado' }
  }
}
