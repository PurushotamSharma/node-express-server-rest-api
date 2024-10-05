pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/rest-api"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = credentials('kubeconfig-credentials-id')
    }
    
    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
        
        stage("Build") {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }
        
        stage("Push to Docker Hub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials-id', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }
        
        stage("Deploy to EKS") {
            steps {
                sh "kubectl --kubeconfig $KUBECONFIG get nodes"
                sh """
                    helm upgrade --install rest-api-release ./helm-chart \
                    --set image.repository=${DOCKER_IMAGE} \
                    --set image.tag=${DOCKER_TAG} \
                    --kubeconfig $KUBECONFIG
                """
            }
        }
    }
    
    post {
        always {
            sh "docker logout"
        }
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed"
            sh "helm --kubeconfig $KUBECONFIG rollback rest-api-release 0 || true"
        }
    }
}
