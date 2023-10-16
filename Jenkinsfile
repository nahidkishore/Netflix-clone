pipeline {
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    
     environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }


    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage("Git Checkout"){
            steps{
               git branch: 'main', url: 'https://github.com/nahidkishore/Netflix-clone.git'
            }
        }
    
         stage("Trivy FS Scan"){
           steps{
               sh "trivy fs . > trivy.txt" 
            }
        }

         stage("OWASP Dependency Check"){
           steps{
                dependencyCheck additionalArguments: '--scan ./' , odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
       stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        
        stage("SonarQube Quality Gate"){
       
             steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
         }

     }
     
     stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        } 

        stage("Build and Push to Docker Hub"){
               steps{
                   
                echo 'login into docker hub and pushing image....'
                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]){
                     sh "docker build . -t netflix-app --build-arg TMDB_V3_API_KEY=c98fdab914b9bacee19db52aeb2707f4"
                     sh "docker tag netflix-app ${env.dockerHubUser}/netflix-app:latest"
                     sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"
                     sh "docker push ${env.dockerHubUser}/netflix-app:latest"


               }
           }
         }

         stage("Deploy to Container"){
            steps{
                sh " docker run -d --name netflix-app -p 8085:80 nahid0002/netflix-app:latest "
            }
        }

          stage("TRIVY"){
            steps{

                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]){
                     
                    sh "trivy image ${env.dockerHubUser}/netflix-app:latest > trivy.txt" 


               }
                
            }
        }

         

        stage('Deploy to kubernetes') {
            steps {
                script {
                   
                    withKubeConfig([credentialsId: 'K8s', serverUrl: '']) {
                        sh ('kubectl apply -f deployment.yaml')
                    }
                }
            }
        }
        

    }


      post {
            always {
                emailext (
                    subject: "Pipeline Status: ${BUILD_NUMBER}",
                    body: '''<html>
                                <body>
                                    <p>Build Status: ${BUILD_STATUS}</p>
                                    <p>Build Number: ${BUILD_NUMBER}</p>
                                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                                </body>
                            </html>''',
                    to: 'nahidkishore99@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
}