pipeline {
  agent any

  // Ajuste os nomes conforme "Manage Jenkins » Tools"
  tools {
    jdk   'jdk-21'     // ex.: JDK 21 instalado no Jenkins
    maven 'maven-3.9'    // ex.: Maven 3.x instalado no Jenkins
  }

   options {
    timestamps()
    ansiColor('xterm')
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
      when { expression { return params.RUN_DEP_SCAN } }
      options { timeout(time: 5, unit: 'MINUTES') } // evita travar o job
      steps {
        // Garante que o cache existe
        bat """
          if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
        """

        // Monta os argumentos (com ou sem API key)
        script {
          def dcArgs = [
            '-DskipTests',
            'org.owasp:dependency-check-maven:check',
            "-DdataDirectory=${env.DC_CACHE}",
            '-Ddependency-check.quickQueryTimestamp=true',
            '-Ddependency-check.cve.validForHours=24',
            // Desative analisadores que você não usa (acelera)
            '-Danalyzers.pg.enabled=false',
            '-Danalyzer.node.audit.enabled=false',
            '-Danalyzer.nuspec.enabled=false',
            '-Danalyzer.assembly.enabled=false'
          ]

          // Se você habilitou a credencial NVD_API_KEY no environment acima:
          if (env.NVD_API_KEY) {
            dcArgs += "-Ddependency-check.nvd.api.key=${env.NVD_API_KEY}"
          }

          bat "mvn -B ${dcArgs.join(' ')}"
        }
      }
      post {
        always {
          // Publica o relatório no Jenkins e arquiva os arquivos
          dependencyCheckPublisher(
            pattern: 'target/dependency-check-report.xml',
            shouldFailBuildOnCVSS: (params.FAIL_CVSS as double),
            stopBuild: false
          )
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true
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