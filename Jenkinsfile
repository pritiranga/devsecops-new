pipeline{
    agent any
    environment {
        DOCKERHUB = "dockerhub-creds"
    }
//    parameters {
//        booleanParam(name: 'enableCleanUp', defaultValue: false, description: 'Select to clean the environements')
//    }
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
                steps {
                	script{	
                        echo "Starting Software Composition Analysis using Dependenct Check Tool..."
 					    dependencyCheck additionalArguments: '--format XML', odcInstallation: 'SCA'
 					    dependencyCheckPublisher pattern: '' 
                    }    
                }
            }
            stage('SonarQube Analysis') {
                //Static Code Analysis using Sonarqube tool
                environment {
                        SCANNER_HOME = tool 'SonarQubeScanner'
                    }
                
                steps {
                    echo "Starting Statis Code Analysis using Sonar Qube Tool..."
                    withSonarQubeEnv(credentialsId: 'sonarqube-token', installationName: 'SonarQubeScanner') {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew sonarqube \
                            -Dsonar.projectKey=TX-DevSecOps-Web \
                            -Dsonar.host.url=http://16.171.181.145:9000 \
                            -Dsonar.login=sqp_1d24d8b51be47fa60aedd19ce7ffb1d1700e3153'
                        } 
                    } 
                }
                stage('Unit Testing') {
                    //Unit Testing using JUnit
                    steps{
                        echo " Starting JUnit Unit tests..."
                            junit(testResults: 'build/test-results/test/*.xml', allowEmptyResults : true, skipPublishingChecks: true)
                    }
                    post {
                        success {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }   
                stage ('Docker File Scan'){
                //Dockerfile Scan using Checkov tool
                    steps{
                        echo "Scanning docker file using CheckOv Tool..."
                        //sh 'pip3 install checkov' 
                        //sh 'docker pull bridgecrew/checkov'
                        sh 'sudo checkov -f Dockerfile --skip-check CKV_DOCKER_3 '        //skip USER in Dockerfile with CKV_DOCKER_3
                    }
                } 



    } //stage closing
} //pipeline closing