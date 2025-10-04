pipeline {
  agent any

  // Ajuste os nomes conforme "Manage Jenkins » Tools"
  tools {
    jdk   'jdk-21'    // ex.: JDK 21 instalado no Jenkins
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

    // ─────────────────────────────────────────────────────────────────────────────
    // PRECHECK: valida a chamada HTTP à API da NVD (sem vazar a chave)
    stage('NVD API precheck') {
      when { expression { return params.RUN_DEP_SCAN } }
      steps {
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
          bat '''
            echo Validando acesso à API da NVD (somente status)...
            powershell -NoProfile -Command ^
              "$h=@{'apiKey'=$env:NVD_API_KEY};" ^
              "try{" ^
              "  $r=Invoke-WebRequest -UseBasicParsing -Headers $h -Uri 'https://services.nvd.nist.gov/rest/json/cves/2.0?resultsPerPage=1';" ^
              "  Write-Host ('NVD HTTP Status: ' + [int]$r.StatusCode)" ^
              "}catch{ if($_.Exception.Response){ $code=[int]$_.Exception.Response.StatusCode; Write-Host ('NVD HTTP Status: ' + $code) } else { Write-Host 'NVD HTTP Status: (sem resposta)'; exit 1 } }"
          '''
        }
      }
    }
    
    stage( 'Vulnerabilidades de verificação de dependência OWASP' ) { 
      steps { 
        dependencyCheck additionalArguments: ''' 
                    -o './' 
                    -s './' 
                    -f 'ALL' 
                    --prettyPrint''' , odcInstallation: 'Vulnerabilidades de verificação de dependência OWASP'
        
         dependencyCheckPublisher pattern: 'dependency-check-report.xml'
       } 
    }
    

  post {
    always {
      echo "Build: ${currentBuild.currentResult}"
    }
  }
}
