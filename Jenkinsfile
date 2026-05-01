pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mi-app'
        IMAGE_TAG  = 'latest'
        SONAR_KEY  = 'cicd-demo'
        SONAR_HOST = 'http://sonarqube:9000'
        APP_HOST_PORT = '80'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "mvn -B sonar:sonar -Dsonar.projectKey=${SONAR_KEY} -Dsonar.host.url=${SONAR_HOST}"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                sh "trivy image --exit-code 1 --severity CRITICAL --no-progress --pkg-types os ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy') {
            when {
                anyOf { branch 'main'; branch 'master' }
            }
            steps {
                sh "docker rm -f ${IMAGE_NAME} || true"
                sh "docker run -d --name ${IMAGE_NAME} -p ${APP_HOST_PORT}:8080 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }

    post {
        success {
            echo "Pipeline OK — imagen ${IMAGE_NAME}:${IMAGE_TAG} desplegada en :${APP_HOST_PORT}"
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
        }
        failure {
            echo "Pipeline FALLO en stage: ${env.STAGE_NAME}"
            sh "docker rm -f ${IMAGE_NAME} || true"
        }
        always {
            echo 'Limpiando workspace...'
            deleteDir()
        }
    }
}
