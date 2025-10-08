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
                ''' // Executa os testes automatizados
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
                }
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                // withCredentials continua igual, para carregar a chave na variável de ambiente
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    
                    // AGORA, usamos um passo 'sh' para executar a ferramenta de linha de comando
                    // O Jenkins garante que a variável ${NVD_API_KEY} seja usada de forma segura aqui
                    sh '''
                        dependency-check.sh --scan "." --format "ALL" --prettyPrint --nvdapiKey "${NVD_API_KEY}" --project "Meu Projeto"
                    '''
                }
                
                // O publisher continua o mesmo, para coletar os resultados após a execução
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
    }

    post {
        always {
            echo "Pipeline finalizado"  // Executado ao final da pipeline, com sucesso ou erro
        }
    }

}
