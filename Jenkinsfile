def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Abdelilahbelhou/pfe_abdelilah.git'
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
                withSonarQubeEnv('sonar-scanner') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=pfe \
                    -Dsonar.projectKey=pfe -Dsonar.java.binaries=.'''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred	'
                }
            }
        }
        
        stage('OWASP SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3') {
                    sh "mvn deploy -X"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t Abde1i1ah01/pfe:latest ."
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html Abde1i1ah01/pfe:latest"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push Abde1i1ah01/pfe:latest"
                    }
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'jk8-cred	', namespace: 'webapps', serverUrl: '') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: '') {
                    sh "ls -la"
                    sh "cat deployment-service.yaml"
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                    sh "kubectl describe svc boardgame-svc"
                }
            }
        }
    }
    
    post {
        always {
            script {
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                slackSend(
                    color: COLOR_MAP.get(pipelineStatus, 'warning'),
                    message: "*${pipelineStatus}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \nMore info at: ${env.BUILD_URL}"
                )
            }
        }
    }
}
