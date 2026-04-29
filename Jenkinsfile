pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mi-app'
        IMAGE_TAG  = 'latest'
        SONAR_KEY  = 'my-app'
        SONAR_HOST = 'http://sonarqube:9000'
        APP_PORT   = '8081'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw -B clean package'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                sh "./mvnw -B sonar:sonar -Dsonar.projectKey=${SONAR_KEY} -Dsonar.host.url=${SONAR_HOST}"
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "trivy image --exit-code 0 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy') {
            when {
                anyOf { branch 'main'; branch 'master' }
            }
            steps {
                sh "docker rm -f ${IMAGE_NAME} || true"
                sh "docker run -d --name ${IMAGE_NAME} -p ${APP_PORT}:8080 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
        }
        always {
            echo 'Limpiando entorno...'
            cleanWs()
        }
    }
}
