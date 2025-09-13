pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
  environment {
        SCANNER_HOME = tool 'sonar'
        SONAR_TOKEN = credentials('sonar-credential')

    }
    stages {
        
        stage('Clean Workspace') {
    steps {
        cleanWs()
    }
}
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Abdelilahbelhou/pfe_abdelilah.git'
            }
        }
        
             stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
            stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=pfe \
                        -Dsonar.projectKey=pfe \
                        -Dsonar.java.binaries=.\
                         -Dsonar.login=$SONAR_TOKEN
          '''
                }
            }
        }
           
         stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-credential'
                }
            }
        }
        

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        
         stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh 'mvn deploy -X'
                }
            }
        }
          
         stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'cred-docker') {
                        sh 'docker build -t abde1i1ah/pfe:latest .'
                    }
                }
            }
        } 
         
         stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html abde1i1ah/pfe:latest'
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'cred-docker') {
                        sh 'docker push abde1i1ah/pfe:latest'
                    }
                }
            }
        }
     

stage('Deploy To Kubernetes'){
    steps{
        withKubeConfig(
            caCertificate: '',
            clusterName: 'docker-desktop',
            contextName: 'docker-desktop',
            credentialsId: 'jenkins-k8s',
            restrictKubeConfigAccess: false,
            serverUrl: 'https://host.docker.internal:62562'
        ){
         sh' kubectl --insecure-skip-tls-verify=true apply -f deployment-service.yaml --validate=false'  
          }
    }
}

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', 
                               credentialsId: 'jenkins-k8s', namespace: 'webapps') {
                   /* sh 'ls -la'
                    sh 'cat deployment-service.yaml'*/
                    sh "kubectl get pods "
                     sh "kubectl get svc "
                }
            }
        }
    }

        
        post {
        always {
            script {
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                slackSend channel: '#pipeline-k8s',
                          color: COLOR_MAP.get(pipelineStatus, 'warning'),
                          message: "${pipelineStatus}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \nMore info at: ${env.BUILD_URL}"
            }
        }
    }
}
