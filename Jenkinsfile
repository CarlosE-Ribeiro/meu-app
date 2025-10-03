pipeline {
  agent any

  tools {
    jdk 'jdk-21'        // mesmo nome configurado em Manage Jenkins > Tools
    maven 'maven-3.9'   // idem
  }

  options { timestamps(); ansiColor('xterm') }

  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build')    { steps { bat 'mvn -B clean package' } }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      echo 'Build finalizado com sucesso ✅'
    }
  }
}
