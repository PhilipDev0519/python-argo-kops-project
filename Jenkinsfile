pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/python-argo"
        DOCKER_TAG   = "latest"
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/<your-username>/python-argo-kops-project.git'
            }
        }

        stage('SonarQube Code Scan') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh """
                      ${SONAR_SCANNER_HOME}/bin/sonar-scanner
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}", "./app")
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                  trivy image \
                  --severity HIGH,CRITICAL \
                  --exit-code 0 \
                  ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                      echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    dockerImage.push("${DOCKER_TAG}")
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
            }
        }
    }

    post {
        success {
            echo "✅ Code quality, security scan, and image push completed. Argo CD will deploy automatically."
        }
        failure {
            echo "❌ Pipeline failed."
        }
    }
}
