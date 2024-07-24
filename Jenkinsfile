pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'   		//'your-aws-region'
        AWS_ACCOUNT_ID = '180899995182'		//'your-aws-account-id'
        ECR_REPOSITORY_1 = 'public.ecr.aws/w2k2d3f8/frontend:latest'	//'your-ecr-repo-1'
        ECR_REPOSITORY_2 = 'public.ecr.aws/w2k2d3f8/backend:latest'	//'your-ecr-repo-2'
        DOCKER_IMAGE_1 = 'frontend'		//'frontend'
        DOCKER_IMAGE_2 = 'backend'		//'backend'
        KUBE_NAMESPACE = 'default'		//'default'
        INGRESS_NAMESPACE = 'ingress-nginx'	//'ingress-nginx'
        CLUSTER_NAME = '3-tier-cluster'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')        
        SLACK_CHANNEL = '3-tier-app-channel'  // Replace with your Slack channel
        SLACK_CREDENTIAL_ID = 'slack-webhook'  // The ID of your Slack webhook credentials in Jenkins
    }

    stages {
        stage('Login to AWS ECR') {
            steps {
                script {
                    sh '''
                    aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin public.ecr.aws/w2k2d3f8
                    '''
                }
            }
        }

        stage('Configure kubectl for EKS') {
            steps {
                script {
                    sh '''
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                    '''
                }
            }
        }

        stage('Build, Tag, and Push Docker Images') {
            steps {
                script {
                    // Build and push the first Docker image
                    sh '''
                    docker build -t ${DOCKER_IMAGE_1} -f ./Docker/frontend/Dockerfile Docker/frontend/
                    docker tag ${DOCKER_IMAGE_1}:latest ${ECR_REPOSITORY_1}
                    docker push ${ECR_REPOSITORY_1}
                    '''

                    // Build and push the second Docker image
                    sh '''
                    docker build -t ${DOCKER_IMAGE_2} -f ./Docker/backend/Dockerfile Docker/backend/
                    docker tag ${DOCKER_IMAGE_2}:latest ${ECR_REPOSITORY_2}
                    docker push ${ECR_REPOSITORY_2}
                    '''
                }
            }
        }

        stage('Deploy Frontend, Backend, and MongoDB') {
            steps {
                script {
                    sh '''
                    kubectl apply -f ./K8s/frontend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f ./K8s/backend --namespace ${KUBE_NAMESPACE}
                    kubectl apply -f ./K8s/mongo-db --namespace ${KUBE_NAMESPACE}

                    '''
                }
            }
        }
        
        stage('Deploy Nginx Ingress Controller') {
            steps {
                script {
                    sh '''
                    kubectl create namespace ${INGRESS_NAMESPACE} || true
                    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
                    helm repo update
                    helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ${INGRESS_NAMESPACE}
                    '''
                }
            }
        }

        stage('Initialize Nginx Ingress Controller') {
            steps {
                sleep time: 20, unit: 'SECONDS'
                echo 'Nginx Ingress Controller initialization complete.'
            }
        }

        stage('Deploy Ingress Resource') {
            steps {
                script {
                    sh '''
                    kubectl apply -f ./K8s/ingress.yaml --namespace ${KUBE_NAMESPACE}
                    '''
                }
            }
        }

        stage('Fetch Load Balancer DNS') {
            steps {
                sleep time: 10, unit: 'SECONDS'
                echo 'Load Balancer initialization complete.'
            }
        }

        stage('Print DNS of Load Balancer') {
            steps {
                script {
                    def ingressName = "main-ingress"
                    def namespace = "${KUBE_NAMESPACE}"
                    def loadBalancerDNS = sh(script: "kubectl get ing ${ingressName} --namespace ${namespace} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    echo "Load Balancer DNS: ${loadBalancerDNS}"
                }
            }
        }
    }

   post {
        success {
            script {
                def ingressName = "main-ingress"
                def namespace = "${KUBE_NAMESPACE}"
                def loadBalancerDNS = sh(script: "kubectl get ing ${ingressName} --namespace ${namespace} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                slackSend(channel: "${SLACK_CHANNEL}", message: "Pipeline completed successfully. Load Balancer DNS: ${loadBalancerDNS}", tokenCredentialId: "${SLACK_CREDENTIAL_ID}")
            }
        }
        failure {
            script {
                slackSend(channel: "${SLACK_CHANNEL}", message: "Pipeline failed. Check the Jenkins logs for details.", tokenCredentialId: "${SLACK_CREDENTIAL_ID}")
            }
        }
    }
}
