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
                    sh "docker run -d --name sanity-test-container ${ECR_REGISTRY}/${REPO_NAME}:v-${BUILD_NUMBER}"
                    sh "sleep 15"
                    try {
                        def containerIp = sh(script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' sanity-test-container", returnStdout: true).trim()
                        sh "curl -v --max-time 10 http://${containerIp}:8080 || (docker logs sanity-test-container && exit 1)"
                    } finally {
                        sh "docker rm -f sanity-test-container"
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