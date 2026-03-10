pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        SONAR_IP     = '43.205.191.220'
        AWS_REGION   = 'ap-south-1'
        ECR_REGISTRY = '386275436648.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_REPO   = "${ECR_REGISTRY}/devsecops-demo"
    }

    stages {

        stage('Trivy FS Scan') {
            steps {
                sh '''
                trivy fs \
                --exit-code 0 \
                --severity HIGH,CRITICAL \
                --skip-dirs target \
                .
                '''
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                    mvn clean verify sonar:sonar \
                    -Dsonar.projectKey=devsecops-demo \
                    -Dsonar.host.url=http://${SONAR_IP}:9000 \
                    -Dsonar.token=${SONAR_TOKEN} \
                    -Dsonar.qualitygate.wait=true
                    """
                }
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | \
                docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build \
                --platform linux/amd64 \
                -t "$IMAGE_REPO:$BUILD_NUMBER" \
                -t "$IMAGE_REPO:latest" \
                .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                "$IMAGE_REPO:$BUILD_NUMBER"
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push "$IMAGE_REPO:$BUILD_NUMBER"'
                sh 'docker push "$IMAGE_REPO:latest"'
            }
        }
    }

    post {
        always {
            cleanWs()
            sh 'docker system prune -af || true'
        }
    }
}
