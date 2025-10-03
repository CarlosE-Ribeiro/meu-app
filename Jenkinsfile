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
    string(name: 'FAIL_CVSS', defaultValue: '0.0', description: 'Falhar build se CVSS >= (ex.: 7.0). Use 0 para nunca falhar.')
  }

  environment {
    // pasta de cache local do Dependency-Check
    DC_CACHE = 'C:\\DC_CACHE'
    // ajustar se quiser falhar o build acima de um certo CVSS
    FAIL_CVSS = '7.0'
    // tunáveis para contornar rate limit/Cloudflare
    NVD_DELAY_MS = '15000'
    NVD_CF_RETRIES = '6'
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

    // 1) Atualiza/baixa a base da NVD para o cache
    stage('OWASP Dependency-Check (update cache)') {
      options { timeout(time: 25, unit: 'MINUTES') } // primeira carga pode demorar
      steps {
        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
          bat """
            if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
            mvn -B org.owasp:dependency-check-maven:9.1.0:updateonly ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -Dnvd.api.delay=%NVD_DELAY_MS% ^
              -Dnvd.api.cloudflare.retries=%NVD_CF_RETRIES% ^
              -DdataDirectory=%DC_CACHE%
          """
        }
      }
    }

    // 2) Roda o scan utilizando o cache baixado
    stage('OWASP Dependency-Check (scan)') {
      options { timeout(time: 12, unit: 'MINUTES') }
      steps {
        bat """
          mvn -B -DskipTests org.owasp:dependency-check-maven:9.1.0:check ^
            -DfailBuildOnCVSS=%FAIL_CVSS% ^
            -Dautoupdate=false ^
            -DdataDirectory=%DC_CACHE% ^
            -Ddependency-check.quickQueryTimestamp=true ^
            -Ddependency-check.cve.validForHours=24 ^
            -Danalyzers.pg.enabled=false ^
            -Danalyzer.node.audit.enabled=false ^
            -Danalyzer.nuspec.enabled=false ^
            -Danalyzer.assembly.enabled=false
        """
      }
      post {
        always {
          // publica no plugin do Jenkins (se instalado)
          dependencyCheckPublisher()
          // guarda relatórios (HTML, XML, JSON)
          archiveArtifacts artifacts: 'target/dependency-check-report.*', allowEmptyArchive: true
        }
      }
    }

  post {
    always {
      echo "Build: ${currentBuild.currentResult}"
    }
  }