pipeline{
    agent any
    environment {
        DOCKERHUB = "dockerhub-creds"
    }
    parameters {
        booleanParam(name: 'enableCleanUp', defaultValue: false, description: 'Select to clean the environements')
    }
    stages {
            stage('Checkout') {
                steps {
                    echo "Checkout the code from GitLab repo..."
                    checkout scm
                }
            }      
            stage ('Software Composition Analysis'){
                //Install OWASP Dependency-Check plugin
                //SCA using Dependency-Check tool
                when {
                    expression{
                        params.enableCleanUp == false
                    }
                }
                steps {
                	script{	
                        echo "Starting Software Composition Analysis using Dependenct Check Tool..."
 					    dependencyCheck additionalArguments: '--format XML', odcInstallation: 'SCA'
 					    dependencyCheckPublisher pattern: '' 
                    }    
                }
            }
    }
}