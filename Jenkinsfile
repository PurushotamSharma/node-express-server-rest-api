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
                echo "Cloning the code"
                git url: "https://github.com/PurushotamSharma/node-express-server-rest-api.git", branch: "master"
                echo "Cloned the code successfully"
            }
        }
        stage("Build") {
            steps {
                echo "Building the code"
                sh "whoami"
                sh "docker build -t rest-api:latest ."
            }
        }
        stage("Push to Docker Hub") {
            steps {
                echo "Pushing the image to Docker Hub"
                withCredentials([usernamePassword(credentialsId: "dockerhubcred", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {
                    sh "echo ${env.dockerHubPass} | docker login -u ${env.dockerHubUser} --password-stdin"
                    sh "docker tag rest-api:latest ${env.dockerHubUser}/rest-api:latest"
                    sh "docker push ${env.dockerHubUser}/rest-api:latest"
                }
            }
        }
        stage("Deploy") {
            steps {
                kubectl apply -f deployment.yml
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed, rolling back to the previous version"
            withKubeConfig([credentialsId: 'aws-eks-kubeconfig', serverUrl: 'https://C7C8E947EF50560DA55D08142769EEDA.gr7.us-east-2.eks.amazonaws.com']) {
                sh "helm rollback rest-api"
            }
        }
    }
}
