pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME       = tool 'sonar-scanner'
        AWS_ACCOUNT_ID     = credentials('aws-account-id')
        AWS_REGION         = 'ap-south-1'
        ECR_REPO           = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/boardgame"
        IMAGE_TAG          = "${BUILD_NUMBER}"
        NEXUS_URL          = 'http://13.127.177.61:8081'
        NEXUS_CREDENTIALS  = 'nexus-credentials'
        SONAR_URL          = 'http://your-sonarqube-url:9000'   // UPDATE THIS
        EKS_CLUSTER_NAME   = 'dev-eks-cluster'                  // UPDATE THIS
        K8S_NAMESPACE      = 'boardgame'
    }

    stages {

        // ─── Stage 1: Checkout ────────────────────────────────────────────
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/ecc-tech/Demo_app.git'
            }
        }

        // ─── Stage 2: Compile ─────────────────────────────────────────────
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        // ─── Stage 3: Unit Tests ──────────────────────────────────────────
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // ─── Stage 4: Code Coverage (JaCoCo) ──────────────────────────────
        stage('Code Coverage') {
            steps {
                sh 'mvn jacoco:report'
            }
        }

        // ─── Stage 5: SonarQube Analysis ──────────────────────────────────
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=Boardgame \
                          -Dsonar.projectName=Boardgame \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -Dsonar.sources=src/main \
                          -Dsonar.tests=src/test
                    """
                }
            }
        }

        // ─── Stage 6: Quality Gate ────────────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─── Stage 7: Build JAR ───────────────────────────────────────────
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        // ─── Stage 8: Publish to Nexus ────────────────────────────────────
        stage('Publish to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${NEXUS_CREDENTIALS}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        mvn deploy \
                          -DskipTests \
                          -Dusername=${NEXUS_USER} \
                          -Dpassword=${NEXUS_PASS}
                    """
                }
            }
        }

        // ─── Stage 9: Docker Build ────────────────────────────────────────
        stage('Docker Build') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest"
            }
        }

        // ─── Stage 10: Push to AWS ECR ────────────────────────────────────
        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-credentials']]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin \
                          ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest
                    """
                }
            }
        }

        // ─── Stage 11: Update K8s Manifest Image Tag ──────────────────────
        stage('Update Manifest') {
            steps {
                sh """
                    sed -i 's|image:.*|image: ${ECR_REPO}:${IMAGE_TAG}|g' deployment-service.yaml
                """
            }
        }

        // ─── Stage 12: Deploy to EKS ──────────────────────────────────────
        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-credentials']]) {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                        kubectl apply -f deployment-service.yaml -n ${K8S_NAMESPACE}
                        kubectl rollout status deployment/boardgame-deployment -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
            }
        }

    }

    post {
        success {
            echo "✅ Pipeline completed successfully! Image: ${ECR_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check stage logs above."
        }
        always {
            // Clean up local docker images to save disk space
            sh "docker rmi ${ECR_REPO}:${IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REPO}:latest || true"
        }
    }
}
