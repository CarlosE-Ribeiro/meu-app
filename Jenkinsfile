pipeline {
    
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }
        
        stage('dependency check'){
            steps {
                dependencyCheck additionalArguments: '', nvdCredentialsId: 'nvd-api-key', odcInstallation: 'OWASP-DC'
            }
        }
    }
}
