pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        AWS_REGION   = 'ap-southeast-1'
        ECR_REPO     = '203848753188.dkr.ecr.ap-southeast-1.amazonaws.com/boardgame'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        CLUSTER_NAME = 'dev-eks-cluster'
        K8S_NS       = 'boardgame'
    }

    stages {

        stage('Compile') {
            steps { sh 'mvn compile' }
        }

        stage('Test') {
            steps { sh 'mvn test' }
            post {
                always { junit '**/target/surefire-reports/*.xml' }
            }
        }

        stage('Build JAR') {
            steps { sh 'mvn clean package -DskipTests' }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                sh "docker tag  ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin \
                          203848753188.dkr.ecr.${AWS_REGION}.amazonaws.com

                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest
                    """
                }
            }
        }

        stage('Update Image in Manifest') {
            steps {
                sh "sed -i 's|image:.*|image: ${ECR_REPO}:${IMAGE_TAG}|g' deployment-service.yaml"
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]]) {
                    sh """
                        aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                        kubectl apply -f deployment-service.yaml -n ${K8S_NS}
                        kubectl rollout status deployment/boardgame-deployment -n ${K8S_NS} --timeout=120s
                    """
                }
            }
        }
    }

    post {
        success { echo "✅ App deployed! Check: kubectl get ingress -n ${K8S_NS}" }
        failure { echo "❌ Pipeline failed. Check stage logs above." }
        always  {
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REPO}:latest || true"
        }
    }
}
