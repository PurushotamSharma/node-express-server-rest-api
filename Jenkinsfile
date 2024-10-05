pipeline{
    agent any
    
    stages{
        stage("Hello"){
            steps{
                script{
                    hello()
                }
            }
        }
        stage("Code"){
            steps{
                echo "This is clonning the code"
                git url : "https://github.com/PurushotamSharma/node-express-server-rest-api.git",branch: "master"
                echo "clonning the code successfully"
            }
        }
        stage("Build"){
            steps{
                echo "This is building the code"
                sh "whoami"
                sh "docker build -t rest-api:latest ."
                
            }
        }
        stage("Push to the Docker Hub"){
            steps{
                echo "This is Pushing the image to docker hub"
                withCredentials([usernamePassword('credentialsId':"dockerhubcred",passwordVariable:"dockerHubPass",usernameVariable:"dockerHubUser")]){
                sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
                sh "docker tag rest-api:latest ${env.dockerHubUser}/rest-api:latest"
                sh "docker push ${env.dockerHubUser}/rest-api:latest"
                }
            }
        }
        stage("Deploy"){
            steps{
                echo "This is deploying the code"
                sh "docker compose up -d "
            }
        }
    }
}
