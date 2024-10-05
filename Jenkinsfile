pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "rest-api"
        DOCKER_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('dockerhubcred')
        EKS_CLUSTER_NAME = "rest-api"
        AWS_REGION = "us-east-2"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
        HELM_CHART_PATH = "/home/ubuntu/rest-api"
        UBUNTU_SERVER = "ubuntu@ec2-18-118-206-196.us-east-2.compute.amazonaws.com"
        DEPLOYMENT_NAME = "rest-api"
        NAMESPACE = "default"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }
        
        stage('Copy Helm Chart') {
            steps {
                script {
                    sshagent(credentials: ['ubuntu-server-ssh-key']) {
                        sh """
                            ssh-keyscan -H ec2-18-118-206-196.us-east-2.compute.amazonaws.com >> ~/.ssh/known_hosts
                            scp -r ${UBUNTU_SERVER}:${HELM_CHART_PATH} ${WORKSPACE}/helm-chart
                        """
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                            aws configure set region ${AWS_REGION}
                            aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME} --kubeconfig ${KUBECONFIG}
                            export KUBECONFIG=${KUBECONFIG}
                            
                            echo "Verifying kubectl configuration:"
                            kubectl config view --minify
                            echo "Listing nodes:"
                            kubectl get nodes
                            
                            echo "Contents of the helm-chart directory:"
                            ls -la ${WORKSPACE}/helm-chart
                            
                            helm upgrade --install ${DEPLOYMENT_NAME} ${WORKSPACE}/helm-chart \
                                --set image.repository=${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE} \
                                --set image.tag=${DOCKER_TAG} \
                                --namespace ${NAMESPACE} \
                                --debug
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE} --timeout=300s
                        """
                    }
                }
            }
        }
        
        stage('Restart Deployment') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            kubectl rollout restart deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
                            kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE} --timeout=300s
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "rm -f ${KUBECONFIG}"
            sh "aws configure set aws_access_key_id ''"
            sh "aws configure set aws_secret_access_key ''"
            sh "docker logout"
        }
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed, rolling back..."
            script {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh """
                        export KUBECONFIG=${KUBECONFIG}
                        helm rollback ${DEPLOYMENT_NAME} 0 -n ${NAMESPACE}
                        kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE} --timeout=300s
                    """
                }
            }
        }
    }
}
