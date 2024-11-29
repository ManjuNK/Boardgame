def COLOR_MAP = [
    'FAILURE' : 'danger',
    'SUCCESS' : 'good'
]
pipeline {
    agent any
    
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    
    environment{
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {     
        stage('Git Checkout') {
            steps {
               git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/ManjuNK/Boardgame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner - Dsonar.projectName=Boardgame -Dsonar.projectKey=Boardgame \
                     -Dsonar.java-binaries=. '''
                }
            }
        }
        
        stage('Sonar Quality Gate') {
            steps {
                script{
                waitForQualityGate abortPipeline: false, credentailsId: 'sonar-token'
                
                }
            }
        }
        
        stage('Maven Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Nexus Push Artifcat') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                  sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Build and tag') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                      sh " docker build -t manjunk/Boardgame:latest ."
                   }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ."
            }
        }
        
        stage('Docker Image push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                      sh " docker push manjunk/Boardgame:latest"
                   }
                }
            }
        }
        
        stage('K8S Deploy Image Scan') {
            steps {
                script{
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.31.4:6443') {
                            sh "kubectl apply -f deployment-service.yaml"
                    }
                }
            }
        }
        
        stage('Verify the K8S Deployment') {
            steps {
                script{
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.31.4:6443') {
                            sh "kubectl get svc -n webapps"
                            sh "kubectl get pods -n webapps"
                    }
                }
            }
        }
    } 
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "nkmanju412@gmail.com";  
            echo 'Slack Notifications'
        slackSend (
            channel: '#jenkins',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} \n build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                )
		    }
    	}
}
