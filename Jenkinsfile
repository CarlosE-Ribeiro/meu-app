// Início do Jenkinsfile
pipeline {
    // 1. Agente: Onde o pipeline vai rodar. 'any' significa qualquer agente disponível.
    agent any

    // 2. Ferramentas: Define quais ferramentas configuradas globalmente serão usadas.
    //    Os nomes aqui (ex: 'jdk-21') devem ser IDÊNTICOS aos da sua tela de "Tools".
    tools {
        jdk   'jdk-21'
        maven 'maven-3.9'
    }

    // 3. Parâmetros: Opções que você pode escolher ao iniciar o build manualmente.
    parameters {
        booleanParam(name: 'RUN_DEP_SCAN', defaultValue: true, description: 'Marque para executar a análise de segurança OWASP.')
    }

    // 4. Variáveis de Ambiente: Definidas para todo o pipeline.
    environment {
        // Define um local para o cache do OWASP dentro do workspace do próprio job.
        DC_CACHE = "${env.WORKSPACE}@tmp\\DC_CACHE"
    }

    // 5. Estágios (Stages): As etapas do nosso processo de CI/CD.
    stages {

        // Estágio 1: Limpeza
        stage('Limpeza do Workspace') {
            steps {
                // Garante que o ambiente está limpo antes de baixar o código.
                echo 'Limpando o workspace antes do build...'
                cleanWs()
            }
        }

        // Estágio 2: Download do Código
        stage('Checkout do Código') {
            steps {
                // Baixa o código do repositório Git configurado na interface do Jenkins.
                echo 'Baixando o código do repositório Git...'
                // Certifique-se de configurar seu repositório na seção "SCM" da UI!
                // Por enquanto, vamos simular com um repositório de exemplo do git.
                git 'https://github.com/exemplo/meu-projeto-java.git'
            }
        }

        // Estágio 3: Compilação
        stage('Build') {
            steps {
                // Usa o Maven para compilar o projeto e criar o pacote (.jar ou .war).
                echo 'Compilando o projeto com Maven...'
                bat 'mvn -B -DskipTests clean package'
            }
            post {
                // Ações pós-estágio
                always {
                    // Arquiva o artefato gerado para que possamos baixá-lo.
                    echo 'Arquivando o artefato gerado...'
                    archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
                }
            }
        }

        // Estágio 4: Testes Unitários
        stage('Testes Unitários') {
            steps {
                // Roda os testes unitários definidos no projeto.
                echo 'Executando testes unitários...'
                bat 'mvn -B test'
            }
            post {
                always {
                    // Publica os resultados dos testes para visualização no Jenkins.
                    echo 'Publicando resultados dos testes...'
                    junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
                }
            }
        }
        
        // Estágio 5: Análise de Segurança
        stage('Análise de Segurança OWASP') {
            // Só executa se o parâmetro 'RUN_DEP_SCAN' estiver marcado como true.
            when {
                expression { params.RUN_DEP_SCAN }
            }
            steps {
                // Usa a credencial da API do NVD de forma segura.
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    script {
                        echo 'Iniciando análise de vulnerabilidades com OWASP Dependency-Check...'
                        // Localiza a ferramenta 'OWASP-DC' configurada globalmente.
                        def dcPath = tool name: 'OWASP-DC', type: 'dependencyCheck'

                        // Executa o comando para Windows.
                        bat """
                            call "${dcPath}\\bin\\dependency-check.bat" --scan "." --format "ALL" --apiKey "%NVD_API_KEY%" --data "%DC_CACHE%"
                        """
                    }
                }
                
                // Publica o relatório do OWASP na página do build.
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
    }

    // 6. Ações Pós-Pipeline: Executadas no final, independentemente do resultado.
    post {
        always {
            echo 'Pipeline finalizado.'
        }
        success {
            // Aqui você poderia adicionar notificações de sucesso (ex: Slack, Email).
            echo 'O pipeline foi executado com sucesso!'
        }
        failure {
            // E aqui, notificações de falha.
            echo 'O pipeline falhou.'
        }
    }
}
// Fim do Jenkinsfile