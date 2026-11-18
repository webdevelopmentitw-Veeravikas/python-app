
pipeline {
    agent any

    environment {
        // AWS Configuration
        AWS_DEFAULT_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '743186641965'

        // ECR Configuration
        ECR_REPOSITORY = 'python-app'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"

        // Docker Configuration
        IMAGE_NAME = 'python-app'

        // EKS Configuration
        EKS_CLUSTER_NAME = 'demo-cluster-1'
       

        // SonarQube Configuration
        SONAR_PROJECT_KEY = 'python-app'
        SONAR_SERVER = 'SonarQube'

        // Kubernetes Manifest
        K8S_MANIFEST = 'pod.yaml'
    }

    stages {

        // 1. GitHub Checkout
        stage('GitHub Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/webdevelopmentitw-Veeravikas/python-app.git'
            }
        }

        // 2. SonarQube Analysis
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv("${SONAR_SERVER}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        // 3. Docker Build
        stage('Docker Build') {
            steps {
                script {
                    env.DOCKER_IMAGE =
                        "${IMAGE_NAME}:${BUILD_NUMBER}"
                }

                sh '''
                    set -e

                    docker build -t "$DOCKER_IMAGE" .

                    docker images
                '''
            }
        }

        // 4. Trivy Image Scan
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -e

                    trivy image \
                    --format table \
                    --severity HIGH,CRITICAL \
                    "$DOCKER_IMAGE" | tee trivy-report.txt
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt',
                                     allowEmptyArchive: true
                }
            }
        }
        
       

        // 5. AWS ECR Push
        stage('ECR Push') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        env.ECR_IMAGE =
                            "${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
                    }

                    sh '''
                        set -e

                        echo "Checking AWS Identity"
                        aws sts get-caller-identity

                        echo "ECR Login"

                        aws ecr get-login-password \
                            --region "$AWS_DEFAULT_REGION" |
                        docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo "Docker Tag"

                        docker tag "$DOCKER_IMAGE" "$ECR_IMAGE"

                        echo "Pushing Image to ECR"

                        docker push "$ECR_IMAGE"

                        echo "ECR Push Successful"
                    '''
                }
            }
        }

        // 6. Deploy to EKS
        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    aws(
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "Updating EKS Kubeconfig"

                        aws eks update-kubeconfig \
                            --region "$AWS_DEFAULT_REGION" \
                            --name "$EKS_CLUSTER_NAME"

                        echo "Checking Kubernetes Nodes"

                        kubectl get nodes

                        echo "Updating Deployment Image"

                        sed -i "s|_IMAGE_|${ECR_IMAGE}|g" "$K8S_MANIFEST"

                        echo "Deploying Application"

                        kubectl apply -f "$K8S_MANIFEST"

                        echo "Checking Deployment"


                        kubectl get pods
                        kubectl get svc

                        echo "Deployment Successful"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Completed Successfully!'
        }

        failure {
            echo 'CI/CD Pipeline Failed! Check Jenkins Console Output.'
        }

        always {
            echo 'Pipeline Execution Finished.'
        }
    }
}
