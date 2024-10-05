pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "rest-api"
        DOCKER_TAG = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = "rest-api"
        AWS_REGION = "us-east-2"
    }
    
    stages {
        stage("Clone Code") {
            steps {
                echo "Cloning the code"
                git url: "https://github.com/PurushotamSharma/node-express-server-rest-api.git", branch: "master"
                echo "Cloned the code successfully"
            }
        }
        
        stage("Build Docker Image") {
            steps {
                echo "Building the Docker image"
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }
        
        stage("Push to Docker Hub") {
            steps {
                echo "Pushing the image to Docker Hub"
                withCredentials([usernamePassword(credentialsId: "dockerhubcred", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {
                    sh "echo \$dockerHubPass | docker login -u \$dockerHubUser --password-stdin"
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} \$dockerHubUser/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push \$dockerHubUser/${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }
        
        stage("Deploy to EKS") {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                        echo "Updating kubeconfig"
                        sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}"
                        
                        echo "Deploying to EKS using Helm"
                        sh """
                            helm upgrade --install rest-api ./helm-chart \
                            --set image.repository=\$dockerHubUser/${DOCKER_IMAGE} \
                            --set image.tag=${DOCKER_TAG} \
                            --namespace default
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
            echo "Deployment failed"
        }
    }
}
