pipeline {
  agent any

  tools {
    jdk 'jdk-21'        // use o mesmo "Name" que você configurou nas Tools
    maven 'maven-3.9'
  }

  options {
    timestamps()
    // ansiColor('xterm')  <-- REMOVIDO
  }

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
