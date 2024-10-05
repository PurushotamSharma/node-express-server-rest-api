pipeline {
    agent any
    
    stages {
        stage("Hello") {
            steps {
                echo "Starting the Pipeline"
            }
        }
        stage("Code") {
            steps {
                echo "This is cloning the code"
                git url: "https://github.com/PurushotamSharma/node-express-server-rest-api.git", branch: "master"
                echo "Cloning the code successfully"
            }
        }
        stage("Build") {
            steps {
                echo "This is building the code"
                sh "whoami"
                sh "docker build -t rest-api:latest ."
            }
        }
        stage("Push to the Docker Hub") {
            steps {
                echo "This is pushing the image to Docker Hub"
                withCredentials([usernamePassword(credentialsId: "dockerhubcred", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {
                    sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
                    sh "docker tag rest-api:latest ${env.dockerHubUser}/rest-api:latest"
                    sh "docker push ${env.dockerHubUser}/rest-api:latest"
                }
            }
        }
        stage("Deploy") {
            steps {
                echo "This is deploying the code to AWS EKS"
                // Note: If you want to specify a namespace, add it here.
                withKubeConfig([credentialsId: 'aws-eks-kubeconfig']) {  
                    // Upgrade or install the Helm release
                    sh "helm upgrade --install rest-api-release ./helm-chart --set image.repository=${env.dockerHubUser}/rest-api --set image.tag=latest --wait --timeout=300s"
                }
            }
        }
    }
    
    post {
        failure {
            echo "Deployment failed, rolling back to the previous version"
            withKubeConfig([credentialsId: 'aws-eks-kubeconfig']) {  
                sh "helm rollback rest-api-release"
            }
        }
    }
}
