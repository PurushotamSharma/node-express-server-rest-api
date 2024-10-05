pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "rest-api"
        DOCKER_TAG = "latest"
    }
    
    stages {
        stage("Initialize") {
            steps {
                echo "Starting the Pipeline"
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                }
            }
        }
        
        stage("Clone Repository") {
            steps {
                echo "Cloning the code"
                git url: "https://github.com/PurushotamSharma/node-express-server-rest-api.git", branch: "master"
                echo "Repository cloned successfully"
            }
        }
        
        stage("Build Docker Image") {
            steps {
                echo "Building Docker image"
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:${env.GIT_COMMIT_SHORT}"
            }
        }
        
        stage("Push to Docker Hub") {
            steps {
                echo "Pushing image to Docker Hub"
                withCredentials([usernamePassword(credentialsId: "dockerhubcred", passwordVariable: "DOCKER_PASSWORD", usernameVariable: "DOCKER_USERNAME")]) {
                    sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_USERNAME}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_USERNAME}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_USERNAME}/${DOCKER_IMAGE}:${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage("Deploy to AWS EKS") {
            steps {
                echo "Deploying to AWS EKS"
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    withKubeConfig([credentialsId: 'aws-eks-kubeconfig', serverUrl: 'https://C7C8E947EF50560DA55D08142769EEDA.gr7.us-east-2.eks.amazonaws.com']) {
                        sh "kubectl config view --raw"
                        sh "kubectl get nodes"
                        
                        echo "Deploying with Helm"
                        sh """
                            helm upgrade --install rest-api-release ./helm-chart \
                            --set image.repository=${DOCKER_USERNAME}/${DOCKER_IMAGE} \
                            --set image.tag=${env.GIT_COMMIT_SHORT} \
                            --wait --timeout=300s
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed, attempting rollback"
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                withKubeConfig([credentialsId: 'aws-eks-kubeconfig', serverUrl: 'https://C7C8E947EF50560DA55D08142769EEDA.gr7.us-east-2.eks.amazonaws.com']) {
                    sh "helm rollback rest-api-release"
                }
            }
        }
        always {
            echo "Cleaning up workspace"
            deleteDir()
        }
    }
}
