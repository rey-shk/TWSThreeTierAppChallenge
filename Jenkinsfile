pipeline {
    agent any

    environment {
        // ==========================================
        // AWS & ECR CONFIGURATION
        // ==========================================
        AWS_ACCOUNT_ID     = '012088228061'
        AWS_REGION         = 'us-east-1'
        AWS_CREDS_ID       = 'aws-ecr-creds'
        
        // ECR Registry URL: <account-id>.dkr.ecr.<region>.amazonaws.com
        ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        BACKEND_REPO       = "aws/backend"
        FRONTEND_REPO      = "aws/frontend"
        
        BACKEND_IMAGE      = "${ECR_REGISTRY}/${BACKEND_REPO}"
        FRONTEND_IMAGE     = "${ECR_REGISTRY}/${FRONTEND_REPO}"
        IMAGE_TAG          = "${env.BUILD_NUMBER}"

        // EKS Cluster Details
        CLUSTER_NAME       = 'three-tier-cluster'
        K8S_NAMESPACE      = 'workshop'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo "Cloning source code from repository..."
                checkout scm
            }
        }

        stage('Code & Security Scan') {
            steps {
                script {
                    echo "Running Trivy file system vulnerability scan..."
                    sh '''
                        if command -v trivy &> /dev/null; then
                            trivy fs --severity HIGH,CRITICAL --format table . || true
                        else
                            echo "Trivy not installed on agent. Skipping filesystem scan."
                        fi
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    echo "Building Backend Docker Image: ${BACKEND_IMAGE}:${IMAGE_TAG}..."
                    sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest ./backend"

                    echo "Building Frontend Docker Image: ${FRONTEND_IMAGE}:${IMAGE_TAG}..."
                    sh "docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest ./frontend"
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                script {
                    echo "Scanning Docker images with Trivy..."
                    sh """
                        if command -v trivy &> /dev/null; then
                            trivy image --severity HIGH,CRITICAL ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                            trivy image --severity HIGH,CRITICAL ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                        else
                            echo "Trivy not installed on agent. Skipping image scan."
                        fi
                    """
                }
            }
        }

        stage('Push Docker Images to AWS ECR') {
            steps {
                script {
                    echo "Authenticating with AWS ECR and pushing images..."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDS_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh """
                            export AWS_ACCESS_KEY_ID=\${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=\${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=${AWS_REGION}

                            # Authenticate Docker client to AWS ECR
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                            # Push Backend Images
                            docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                            docker push ${BACKEND_IMAGE}:latest

                            # Push Frontend Images
                            docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            docker push ${FRONTEND_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Updating Kubernetes manifests and deploying to EKS cluster..."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDS_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh """
                            export AWS_ACCESS_KEY_ID=\${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=\${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=${AWS_REGION}

                            # Update kubeconfig for EKS
                            if command -v aws &> /dev/null; then
                                aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME} || true
                            fi

                            # Ensure target namespace exists
                            kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                            # 1. Deploy MongoDB secret, service and deployment
                            echo "Deploying MongoDB..."
                            kubectl apply -f k8s_manifests/mongo/secrets.yaml -n ${K8S_NAMESPACE}
                            kubectl apply -f k8s_manifests/mongo/service.yaml -n ${K8S_NAMESPACE}
                            kubectl apply -f k8s_manifests/mongo/deploy.yaml -n ${K8S_NAMESPACE}

                            # 2. Update Image tags in Backend & Frontend manifests
                            sed -i "s|image:.*|image: ${BACKEND_IMAGE}:${IMAGE_TAG}|g" k8s_manifests/backend-deployment.yaml
                            sed -i "s|image:.*|image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|g" k8s_manifests/frontend-deployment.yaml

                            # 3. Deploy Backend
                            echo "Deploying Backend API..."
                            kubectl apply -f k8s_manifests/backend-service.yaml -n ${K8S_NAMESPACE}
                            kubectl apply -f k8s_manifests/backend-deployment.yaml -n ${K8S_NAMESPACE}

                            # 4. Deploy Frontend
                            echo "Deploying Frontend UI..."
                            kubectl apply -f k8s_manifests/frontend-service.yaml -n ${K8S_NAMESPACE}
                            kubectl apply -f k8s_manifests/frontend-deployment.yaml -n ${K8S_NAMESPACE}

                            # 5. Apply Ingress / Load Balancer (if applicable)
                            if [ -f k8s_manifests/full_stack_lb.yaml ]; then
                                echo "Deploying Ingress..."
                                kubectl apply -f k8s_manifests/full_stack_lb.yaml -n ${K8S_NAMESPACE} || true
                            fi
                        """
                    }
                }
            }
        }

        stage('Verify Rollout Status') {
            steps {
                script {
                    echo "Verifying deployment rollouts in namespace '${K8S_NAMESPACE}'..."
                    sh """
                        kubectl rollout status deployment/mongodb -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl rollout status deployment/api -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl rollout status deployment/frontend -n ${K8S_NAMESPACE} --timeout=120s

                        echo "=== Deployed Pods ==="
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide

                        echo "=== Deployed Services ==="
                        kubectl get svc -n ${K8S_NAMESPACE}

                        echo "=== Deployed Ingress / Load Balancer ==="
                        kubectl get ingress -n ${K8S_NAMESPACE} || true
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up local Docker images to conserve disk space..."
                sh """
                    docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                    docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                """
            }
        }
        success {
            echo "Pipeline completed successfully! Application is live on Kubernetes."
        }
        failure {
            echo "Pipeline failed! Inspect console output for detailed logs."
        }
    }
}
