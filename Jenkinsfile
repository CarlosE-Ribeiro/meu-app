pipeline {
  agent any
  tools { jdk 'jdk-21'; maven 'maven-3.9' }
  options { timestamps() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build') { steps { bat 'mvn -B -DskipTests clean package' } }

    stage('Test') {
      steps { bat 'mvn -B test' }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }

    stage('OWASP Dependency-Check') {
      steps {
        bat 'mvn -B -DskipTests org.owasp:dependency-check-maven:check'
      }
      post {
        always {
          publishHTML(target: [
            reportDir: 'target',
            reportFiles: 'dependency-check-report.html',
            reportName: 'OWASP Dependency-Check',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: false
          ])
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true
        }
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
  }
}
