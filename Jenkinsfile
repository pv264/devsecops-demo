pipeline {
agent any

```
tools {
    maven 'maven3'
}

environment {
    SONAR_IP     = '13.206.12.244'
    AWS_REGION   = 'ap-south-1'
    ECR_REGISTRY = '513348750131.dkr.ecr.ap-south-1.amazonaws.com'
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
            withCredentials([string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')]) {
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
            --exit-code 0 \
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

    stage('Update Helm Repo') {
        steps {
            withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh '''
                rm -rf devsecops-demo-helm || true

                git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/pv264/devsecops-demo-helm.git
                cd devsecops-demo-helm/devsecops-demo

                yq e ".image.tag = \\"${BUILD_NUMBER}\\"" -i values.yaml

                git config user.email "jenkins@demo.com"
                git config user.name "jenkins"

                git add values.yaml
                git commit -m "Update image tag to ${BUILD_NUMBER}" || true
                git push origin main
                '''
            }
        }
    }

}

post {
    success {
        echo "Pipeline completed successfully"
    }
    failure {
        echo "Pipeline failed"
    }
}
```

}
