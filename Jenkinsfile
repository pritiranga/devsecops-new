pipeline{
    agent any
    environment {
        DOCKERHUB = "dockerhub-creds"
        DOCKERHUB_USER = "pritidevops"
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
                            -Dsonar.host.url=http://16.170.225.76:9000 \
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
            stage('Build'){
                //Building Docker Image
                steps{
                    echo "Building the docker file..."
                    sshagent(['ssh']){
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@16.170.225.76 "export PATH=\$PATH:/opt/gradle/gradle-7.1.1/bin && cd devsecops && docker build -t devsecops . && docker tag devsecops:latest $DOCKERHUB_USER/devsecops:latest"'
                    }
                }   
            } 
            stage('Image Scanning') {
                //Image scanning using Trivy tool
		        steps{
                    echo "Scanning the docker image using Trivy..."
                    sshagent(['ssh']){
                        sh 'ssh -o StrictHostKeyChecking=no ubuntu@16.170.225.76 "trivy image $DOCKERHUB_USER/devsecops:latest"'
                    }
		        }
	        }  
            stage('Publishing Images to Dockerhub') {
                steps {
                    echo "Pushing the image created to Dockerhub..."
                    sshagent(['ssh']) {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            sh '''
                            ssh -o StrictHostKeyChecking=no ubuntu@16.170.225.76 "
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin &&
                            docker push $DOCKERHUB_USER/devsecops:latest &&
                            docker rmi -f devsecops:latest &&
                            docker rmi -f $DOCKERHUB_USER/devsecops:latest"
                            '''
                        }
                    }
                }
            }
            stage('Creating Environments'){
                // Creating namespaces for different environments on k8 cluster
                steps{
                    echo "Preping staging and Production environment..."
                    sshagent(['k8-config']){
                        sh 'ssh -o StrictHostKeyChecking=no ubuntu@13.60.248.189 "kubectl create ns staging && kubectl create ns prod"'
                    }
                }   
            }
                stage('Staging Deployment'){
                //Application deploying on Staging server
                steps{
                    echo "Deploy image to Staging environment..."
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@13.60.248.189 "kubectl create secret docker-registry regcred --docker-server=192.168.6.99:8082 --docker-username=$DOCKER_USERNAME --docker-password=$DOCKER_PASSWORD -n staging && kubectl create secret docker-registry regcred --docker-server=192.168.6.99:8082 --docker-username=$DOCKER_USERNAME --docker-password=$DOCKER_PASSWORD -n prod"
                        """
                    }
                    kubernetesDeploy(
                        configs: 'k8-staging.yml',
                        kubeconfigId: 'k8-config',
                        enableConfigSubstitution: true 
                    )
                }
            }
            stage('Web Application Scanning'){
                //Web Application scanning using ZAP        
                when {
                    expression{
                        params.enableCleanUp == false
                    }
                }
                steps{
                    echo "Performing DAST Scan on app using ZAP tool..."
                    catchError(buildResult: 'Success', stageResult: 'Success'){
                    sleep time: 30, unit: 'SECONDS'
                    sshagent(['dev'])
                    {
                        sh 'ssh -o StrictHostKeyChecking=no testing@192.168.6.99 "docker pull owasp/zap2docker-stable && docker run -t owasp/zap2docker-stable zap-baseline.py -t http://192.168.6.68:32000/VulnerableApp/ && docker rmi -f owasp/zap2docker-stable"'
                    }
                    }
                }
            }
            stage('Production Approval'){
                //Approval for deployment on production environment
                when {
                    expression{
                        params.enableCleanUp == false
                    }
                }
                steps{              
                    script {
                        timeout(time: 10, unit: 'MINUTES'){
                            input ('Deploy to Production?')
                        }
                    }        
                } 
            }
            stage('Production Deployment'){
                //Application deploying on production server
                when {
                    expression{
                        params.enableCleanUp == false
                    }
                }
                steps{
                    script{
                        echo "Deploy image to Production environment..."
                        kubernetesDeploy(
                            configs: 'k8-prod.yml',
                            kubeconfigId: 'k8-cred',
                            enableConfigSubstitution: true 
                        )
                    }
                }
            }



            




    } //stage closing
} //pipeline closing