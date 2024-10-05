pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "rest-api"
        DOCKER_TAG = "latest"
    }
    
    stages {
        stage("Hello") {
            steps {
                echo "Starting the Pipeline"
            }
        }
        
        stage("Code") {
            steps {
                echo "Cloning the code"
                git url: "https://github.com/PurushotamSharma/node-express-server-rest-api.git", branch: "master"
                echo "Cloned the code successfully"
            }
        }
        
        stage("Build") {
            steps {
                echo "Building the code"
                sh "whoami"
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
        
        stage("Helm Deploy") {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh "aws eks update-kubeconfig --name sample --region us-east-2"
                        sh "kubectl get ns"
                        sh "helm upgrade --install restapi ./rest-api --set image.tag=${DOCKER_TAG}"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed, rolling back to the previous version"
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                sh "helm rollback restapi"
            }
        }
    }
}
