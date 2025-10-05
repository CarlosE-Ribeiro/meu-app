pipeline {

 agent any



 // Ajuste os nomes conforme "Manage Jenkins » Tools"

tools {

 jdk 'jdk-21' // ex.: JDK 21 instalado no Jenkins

 maven 'maven-3.9' // ex.: Maven 3.x instalado no Jenkins

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

// pasta de cache local do Dependency-Check

DC_CACHE = 'C:\\DC_CACHE'

// tunáveis para contornar rate limit/Cloudflare

NVD_DELAY_MS = '30000'

NVD_RETRIES = '15'

NVD_CF_RETRIES = '15'

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



stage('OWASP Dependency-Check') {
  when { expression { return params.RUN_DEP_SCAN } }
  steps {
    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
      bat 'if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"'

      dependencyCheck(
        odcInstallation: 'OWASP-DC',          // <-- MESMO nome configurado em Manage Jenkins → Tools
        dataDirectory: "${env.DC_CACHE}",
        additionalArguments: """
          --scan .
          --format ALL
          --prettyPrint
          --failOnCVSS ${params.FAIL_CVSS}
          --nvdApiDelay ${env.NVD_DELAY_MS}
          --nvdApiRetries ${env.NVD_RETRIES}
          --nvdApiCloudflareRetries ${env.NVD_CF_RETRIES}
          --disableAssembly
          --disableNodeJS
          --disableNodeAudit
          --disableNugetConf
          --disablePnpmAudit
          --disableYarnAudit
        """.trim().replaceAll("\\s+"," ")
      )

      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'dependency-check-report.*', allowEmptyArchive: true
    }
  }
}



 }

 post {

 always {

echo "Pipeline finalizado"

 }

  }

}