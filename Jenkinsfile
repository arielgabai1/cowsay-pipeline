pipeline {
    agent any

    triggers {
        pollSCM("* * * * *")
    }

    options {
        timeout(time: 5, unit: "MINUTES")
        timestamps()
    }

    environment {
        SSH_CRED_ID = 'deploy-server-key'
        REPO_NAME = 'cowsay_pipeline'
        ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
        IMAGE = "${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER}"
    }

    stages {
        stage('Build Image') {
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }

        stage('Sanity Test') {
            steps {
                sh "docker rm -f sanity-test-container || true"
                sh "docker run -d --name sanity-test-container ${IMAGE}"
                sh "sleep 15"
                sh "docker exec sanity-test-container node -e 'require(\"http\").get(\"http://127.0.0.1:8080\", (res) => res.pipe(process.stdout))'"
            }
            post {
                always {
                    sh "docker rm -f sanity-test-container || true"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                sh "docker push ${IMAGE}"
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    sh "scp -o StrictHostKeyChecking=no -i $SSH_KEY docker-compose.yml ${SSH_USER}@${env.DEPLOY_SERVER_IP}:/home/${SSH_USER}/docker-compose.yml"
                    sh """
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY ${SSH_USER}@${env.DEPLOY_SERVER_IP} '
                            aws ecr get-login-password --region ${env.AWS_REGION} | sudo docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            echo "ECR_IMAGE=${IMAGE}" > .env
                            sudo docker-compose down || true
                            sudo docker-compose pull
                            sudo docker-compose up -d
                        '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE} || true"
        }
    }
}