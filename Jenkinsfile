pipeline {
    agent any
    tools {
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        REPOSITORY = '192.168.222.60:8083'
        IMAGE_REPO = 'python-system-monitoring'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    stages {
        stage ('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage ('Checkout From Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Louis-2b/DevOps-CICD-Python-Webapp.git'
            }
        }
        stage ('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Python-Webapp \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Python-Webapp 
                    '''
                }
            }
        }
        stage ('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SONAR-TOKEN'
                }
            }
        }
        stage ('TRIVY File scan') {
            steps {
                sh "trivy fs . > trivy-fs_report.txt"
            }
        }
        stage ('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ('Docker Build') {
            steps {
                script {
                    docker.withRegistry("http://${REPOSITORY}", 'NEXUS-TOKEN') {
                        sh "make image"
                    }
                }
            }
        }
        stage ('TRIVY') {
            steps {
                sh "trivy image ${REPOSITORY}/${IMAGE_REPO}:${IMAGE_TAG} > trivy.txt"
            }
        }
        stage ('Docker Push') {
            steps {
                script {
                    docker.withRegistry("http://${REPOSITORY}", 'NEXUS-TOKEN') {
                        sh "make push"
                    }
                }
            }
        }
        stage ("Deploy to container") {
            steps {
                sh "docker run -d --name python1 -p 5000:5000 ${REPOSITORY}/${IMAGE_REPO}:${IMAGE_TAG}"
            }
        }
        stage ('Deploy to kubernetes') {
            steps {
                dir ('kubernetes/development') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
                dir ('kubernetes/staging') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'stephanetchanga2b@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}