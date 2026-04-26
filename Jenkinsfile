pipeline {
    agent any

    options {
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        // Usamos el nombre del servicio 'nexus' definido en docker-compose
        NEXUS_URL = "http://nexus:8081" 
        NEXUS_DOCKER_REGISTRY = "nexus:8083"
        CREDENTIALS_ID = "nexus-credentials"
        IMAGE_NAME = "sumador"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        // Forzar versión de API para compatibilidad con host nuevo
        DOCKER_API_VERSION = "1.44"
    }

    stages {
        stage('Security - Dependency Scan') {
            steps {
                echo "Running npm audit..."
                sh "npm audit || true" 
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Run tests') {
            steps {
                echo "Running tests inside container..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

        stage('Security - Image Scan (Trivy)') {
            steps {
                echo "Scanning image with Trivy..."
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
        
        stage('Tag Docker Image') {
            steps {
                echo "Tagging Docker image for Nexus..."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: CREDENTIALS_ID, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        // Importante: Debes haber añadido '127.0.0.1 nexus' a tu archivo hosts de Windows
                        sh "docker login -u ${USER} -p ${PASS} ${NEXUS_DOCKER_REGISTRY}"
                        sh "docker push ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up..."
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
