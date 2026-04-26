pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        NEXUS_URL = "http://nexus:8081" 
        NEXUS_DOCKER_REGISTRY = "nexus:8083"
        CREDENTIALS_ID = "nexus-credentials"
        IMAGE_NAME = "sumador"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        DOCKER_API_VERSION = "1.44"
    }

    stages {
        stage('Security - Dependency Scan') {
            steps {
                echo "Running npm audit..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm audit --audit-level=critical"
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
                sh """
                    docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image \
                    --severity CRITICAL \
                    --exit-code 1 \
                    ${IMAGE_NAME}:${IMAGE_TAG}                """
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
                withCredentials([usernamePassword(
                    credentialsId: CREDENTIALS_ID, 
                    usernameVariable: 'NEXUS_USER', 
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo \${NEXUS_PASS} | docker login ${NEXUS_DOCKER_REGISTRY} -u \${NEXUS_USER} --password-stdin
                        docker push ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker logout ${NEXUS_DOCKER_REGISTRY}
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${NEXUS_DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true
            """
        }
        success {
            echo "Pipeline completed successfully! Image is in Nexus."
        }
        failure {
            echo "Pipeline failed. Check the logs (possibly a Trivy vulnerability found)."
        }
    }
}
