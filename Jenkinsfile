pipeline {
    agent any

    environment {
        // ==========================================
        // CONFIGURATION PARAMETERS
        // ==========================================
        // Change this to your Docker Hub username (e.g., 'johndoe')
        DOCKER_HUB_USER     = 'your-dockerhub-username'
        
        // Jenkins Credentials ID for Docker Hub (Username with Password)
        DOCKER_CREDS_ID     = 'dockerhub-creds'
        
        // Image names and dynamic build tag
        BACKEND_IMAGE_NAME  = "${DOCKER_HUB_USER}/three-tier-backend"
        FRONTEND_IMAGE_NAME = "${DOCKER_HUB_USER}/three-tier-frontend"
        IMAGE_TAG           = "${env.BUILD_NUMBER}"

        // AWS / Kubernetes configuration
        AWS_REGION          = 'us-west-2'
        CLUSTER_NAME        = 'three-tier-cluster'
        K8S_NAMESPACE       = 'workshop'
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
                    echo "Building Backend Docker Image: ${BACKEND_IMAGE_NAME}:${IMAGE_TAG}..."
                    sh "docker build -t ${BACKEND_IMAGE_NAME}:${IMAGE_TAG} -t ${BACKEND_IMAGE_NAME}:latest ./backend"

                    echo "Building Frontend Docker Image: ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG}..."
                    sh "docker build -t ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} -t ${FRONTEND_IMAGE_NAME}:latest ./frontend"
                }
            }
        }

        stage('Image Security Scan') {
            steps {
                script {
                    echo "Scanning Docker images with Trivy..."
                    sh '''
                        if command -v trivy &> /dev/null; then
                            trivy image --severity HIGH,CRITICAL ${BACKEND_IMAGE_NAME}:${IMAGE_TAG} || true
                            trivy image --severity HIGH,CRITICAL ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} || true
                        else
                            echo "Trivy not installed on agent. Skipping image scan."
                        fi
                    '''
                }
            }
        }

        stage('Push Docker Images to Registry') {
            steps {
                script {
                    echo "Logging into Docker Hub and pushing images..."
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh 'echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin'
                        
                        sh "docker push ${BACKEND_IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${BACKEND_IMAGE_NAME}:latest"
                        
                        sh "docker push ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${FRONTEND_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Updating Kubernetes manifests and deploying to EKS cluster..."
                    sh """
                        # Update kubeconfig for EKS (if using AWS CLI with IAM permissions)
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
                        sed -i "s|image:.*|image: ${BACKEND_IMAGE_NAME}:${IMAGE_TAG}|g" k8s_manifests/backend-deployment.yaml
                        sed -i "s|image:.*|image: ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG}|g" k8s_manifests/frontend-deployment.yaml

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
                    docker rmi ${BACKEND_IMAGE_NAME}:${IMAGE_TAG} || true
                    docker rmi ${FRONTEND_IMAGE_NAME}:${IMAGE_TAG} || true
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
