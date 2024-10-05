pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "purushotamsharma/rest-api"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
        
        stage("Build") {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }
        
        stage("Push to Docker Hub") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhubcred', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }
        
        stage("Deploy to EKS") {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: 'us-east-2') {
                        sh "aws eks get-token --cluster-name rest-api | kubectl apply -f -"
                        sh """
                            helm upgrade --install rest-api-release ./helm-chart \
                            --set image.repository=${DOCKER_IMAGE} \
                            --set image.tag=${DOCKER_TAG}
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh "docker logout"
            }
        }
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed"
            script {
                withAWS(credentials: 'aws-credentials', region: 'us-east-2') {
                    sh "helm rollback rest-api-release 0 || true"
                }
            }
        }
    }
}
