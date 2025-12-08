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
    }


    stages {
        stage('Build Image') {
            steps {
                script {
                    sh "docker build -t ${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER} ."
                }
            }
        }

       stage('Sanity Test') {
            steps {
                script {
                    echo "Running simplified Sanity Test..."
                    sh "docker rm -f sanity-test || true"
                    sh "docker run -d -p 8081:8080 --name sanity-test ${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER}"
                    sh "sleep 15"
                    try {
                        echo "Checking application..."
                        sh "curl -f http://localhost:8081"
                    } finally {
                        sh "docker rm -f sanity-test"
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker push ${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: SSH_CRED_ID, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    script {
                        def sshOptions = "-o StrictHostKeyChecking=no -i $SSH_KEY"
                        
                        sh "scp ${sshOptions} docker-compose.yaml ${SSH_USER}@${env.DEPLOY_SERVER_IP}:/home/${SSH_USER}/docker-compose.yaml"
                        
                        sh """
                        ssh ${sshOptions} ${SSH_USER}@${env.DEPLOY_SERVER_IP} '
                            aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            export ECR_IMAGE_FULL=${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER}
                            docker-compose down || true
                            docker-compose pull
                            docker-compose up -d
                        '
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER} || true"
        }
    }
}