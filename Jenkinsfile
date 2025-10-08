pipeline {
    agent any

    // Ajuste os nomes conforme "Manage Jenkins » Tools"
    tools {
        jdk 'jdk-21'            // ex.: JDK 21 instalado no Jenkins
        maven 'maven-3.9'       // ex.: Maven 3.x instalado no Jenkins
    }

    options {
        timestamps()                                // Adiciona timestamps nos logs
        buildDiscarder(logRotator(numToKeepStr: '20'))  // Mantém as últimas 20 builds
    }

    parameters {
        booleanParam(
            name: 'RUN_DEP_SCAN', 
            defaultValue: true, 
            description: 'Executar OWASP Dependency-Check?'
        )
        string(
            name: 'FAIL_CVSS', 
            defaultValue: '7.0', 
            description: 'Falhar build se CVSS >= (ex.: 7.0). Use 0 para nunca falhar.'
        )
    }

    environment {
        DC_CACHE       = 'C:\\DC_CACHE'  // Diretório de cache para o Dependency-Check
        NVD_DELAY_MS   = '30000'         // Delay entre requisições
        NVD_RETRIES    = '15'            // Tentativas em caso de erro
        NVD_CF_RETRIES = '15'            // Tentativas adicionais com Cloudflare
    }

    stages {


        stage('Build') {
            steps {
                bat '''
                    mvn -B -DskipTests clean package
                ''' // Compila o projeto sem rodar os testes
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
                rem gera o relatorio HTML dos testes
                mvn -B surefire-report:report -Daggregate=true
                '''
            }
            post {
                always {
                // tabelas e gráficos nativos do Jenkins (histórico, etc.)
                junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'

                // publica o HTML bonito
                publishHTML(target: [
                    allowMissing: true,
                    keepAll: true,
                    reportDir: 'target/site',
                    reportFiles: 'surefire-report.html',
                    reportName: 'Test Report (Surefire HTML)'
                ])
                }
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                // se estiver usando cache offline, adicione -Dnoupdate
                bat """
                if not exist "%DC_CACHE%" mkdir "%DC_CACHE%"
                mvn -B org.owasp:dependency-check-maven:check ^
                    -Dformat=HTML,XML ^
                    -Ddata="%DC_CACHE%" ^
                    -DfailOnCVSS=%FAIL_CVSS% ^
                    -DfailOnError=false
                """
            }
            post {
                always {
                // publica no painel padrão do plugin OWASP (link "Dependency-Check")
                dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'

                // publica o HTML bonito
                publishHTML(target: [
                    allowMissing: true,
                    keepAll: true,
                    reportDir: 'target',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency-Check (HTML)'
                ])

                // tabela integrada (Warnings NG) - opcional, mas recomendo:
                recordIssues enabledForFailure: true, tools: [
                    dependencyCheck(pattern: 'target/dependency-check-report.xml')
                ]
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finalizado"  // Executado ao final da pipeline, com sucesso ou erro
        }
    }

}
