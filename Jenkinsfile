pipeline {
  agent any

  // Ajuste os nomes conforme "Manage Jenkins » Tools"
  tools {
    jdk   'JDK21'     // ex.: JDK 21 instalado no Jenkins
    maven 'Maven3'    // ex.: Maven 3.x instalado no Jenkins
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  stages {

    stage('Checkout') {
      steps {
        // Se o job já aponta para seu repo, "checkout scm" basta
        checkout scm
      }
    }

    stage('Build') {
      steps {
        // build sem testes; empacota o JAR
        bat 'mvn -B -DskipTests clean package'
      }
      post {
        success {
          // arquiva o jar para download no Jenkins
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('Test') {
      steps {
        // executa testes unitários
        bat 'mvn -B test'
      }
      post {
        always {
          // publica resultados JUnit (aparece como "Test Result")
          junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
      }
    }

    stage('OWASP Dependency-Check') {
      steps {
        // roda o scan (sem testes)
        bat 'mvn -B -DskipTests org.owasp:dependency-check-maven:check'
      }
      post {
        always {
          // Publisher nativo do plugin "OWASP Dependency-Check"
          // -> mostra painel com vulnerabilidades + tendência
          dependencyCheckPublisher(
            pattern: 'target/dependency-check-report.xml',
            // Política: não falhar o build por CVSS (ajuste se quiser)
            shouldFailBuildOnCVSS: 0.0,
            stopBuild: false
          )

          // (Opcional) Relatório HTML bonitinho – requer plugin "HTML Publisher"
          // Descomente se quiser o link de HTML no build:
          /*
          publishHTML(target: [
            reportDir: 'target',
            reportFiles: 'dependency-check-report.html',
            reportName: 'OWASP Dependency-Check',
            keepAll: true,
            alwaysLinkToLastBuild: true
          ])
          */

          // Arquiva os relatórios para download
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finalizado: ${currentBuild.currentResult}"
    }
  }
}
