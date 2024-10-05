pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "rest-api"
        DOCKER_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('dockerhubcred')
        EKS_CLUSTER_NAME = "rest-api"
        AWS_REGION = "us-east-2"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
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
                            
                            echo "Contents of the current directory:"
                            ls -la
                            
                            CHART_DIR=\$(find . -name Chart.yaml -print0 | xargs -0 -n1 dirname | head -n1)
                            if [ -n "\$CHART_DIR" ]; then
                                echo "Helm chart found in: \$CHART_DIR"
                                echo "Contents of Helm chart directory:"
                                ls -la \$CHART_DIR
                                
                                helm upgrade --install rest-api \$CHART_DIR \
                                    --set image.repository=${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_IMAGE} \
                                    --set image.tag=${DOCKER_TAG} \
                                    --namespace default \
                                    --debug
                            else
                                echo "No Helm chart found in the repository"
                                exit 1
                            fi
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
            echo "Deployment failed"
        }
    }
}
