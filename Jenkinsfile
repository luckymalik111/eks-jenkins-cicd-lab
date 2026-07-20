pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID = '265243687024'
        AWS_DEFAULT_REGION = 'us-east-1'
        ECR_REPO = 'eks-cicd-lab-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPO}"
        S3_BUCKET = 'eks-cicd-lab-reports-265243687024'
        EKS_CLUSTER = 'eks-cicd-lab-cluster'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${ECR_REPO}:${IMAGE_TAG} .'
            }
        }
        stage('Login to ECR') {
            steps {
                sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com'
            }
        }
        stage('Push to ECR') {
            steps {
                sh '''
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:latest
                    docker push ${ECR_URI}:${IMAGE_TAG}
                    docker push ${ECR_URI}:latest
                '''
            }
        }
        stage('Deploy to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${EKS_CLUSTER}
                    sed -i "s|IMAGE_PLACEHOLDER|${ECR_URI}:${IMAGE_TAG}|g" k8s/deployment.yaml
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                    kubectl rollout status deployment/eks-cicd-lab-app --timeout=120s
                '''
            }
        }
        stage('Save Report to S3') {
            steps {
                sh '''
                    echo "{\"build\": \"${BUILD_NUMBER}\", \"image\": \"${ECR_URI}:${IMAGE_TAG}\", \"status\": \"SUCCESS\", \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" > report.json
                    aws s3 cp report.json s3://${S3_BUCKET}/reports/build-${BUILD_NUMBER}.json
                '''
            }
        }
    }
    post {
        always {
            sh 'docker rmi ${ECR_REPO}:${IMAGE_TAG} || true'
        }
    }
}
