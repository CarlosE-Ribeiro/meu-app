pipeline {
  agent any

  // Ajuste os nomes conforme "Manage Jenkins » Tools"
  tools {
    jdk   'jdk-21'      // ex.: JDK 21 instalado no Jenkins
    maven 'maven-3.9'   // ex.: Maven 3.x instalado no Jenkins
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
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
        bat """
            if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
            mvn -B org.owasp:dependency-check-maven:9.1.0:update-only ^
            -Dnvd.api.key=%NVD_API_KEY% ^
            -Dnvd.api.delay=%NVD_DELAY_MS% ^
            -Dnvd.api.retries=10 ^
            -Dnvd.api.cloudflare.retries=10 ^
            -DdataDirectory=%DC_CACHE% ^
            -Dhttps.protocols=TLSv1.2,TLSv1.3
            /* Se você usa proxy corporativo, descomente as 2 linhas abaixo:
            ^ -Dhttps.proxyHost=proxy.seudominio.local ^
            -Dhttps.proxyPort=3128
            */
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
        dependencyCheckPublisher()
        archiveArtifacts artifacts: 'target/dependency-check-report.*', allowEmptyArchive: true
        }
      }
    }

  } // <----- ADICIONADA: fecha o bloco stages

  post {
    always {
      echo "Build: ${currentBuild.currentResult}"
    }
  }

} // <----- ADICIONADA: fecha o bloco pipeline
