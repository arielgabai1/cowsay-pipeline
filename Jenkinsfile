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
                sh "docker rm -f sanity-test || true"
                sh "docker run -d --name sanity-test ${IMAGE}"
                sh "sleep 5"
                sh "docker exec sanity-test curl -f http://127.0.0.1:8080"
            }
            post {
                always {
                    sh "docker rm -f sanity-test || true"
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
                    sh "scp -o StrictHostKeyChecking=no -i \$SSH_KEY docker-compose.yml ${SSH_USER}@${env.DEPLOY_SERVER_IP}:/home/${SSH_USER}/docker-compose.yml"
                    sh """
                        ssh -o StrictHostKeyChecking=no -i \$SSH_KEY ${SSH_USER}@${env.DEPLOY_SERVER_IP} '
                            aws ecr get-login-password --region ${env.AWS_REGION} | sudo docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            echo "ECR_IMAGE=${IMAGE}" > .env
                            sudo docker-compose down || true
                            sudo docker-compose pull
                            sudo docker-compose up -d --remove-orphans
                            sudo docker system prune -a -f
                        '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE} || true"
            sh "docker system prune -f || true"
        }
        success {
            script {
                def authorEmail = sh(returnStdout: true, script: "git show -s --format='%ae'").trim()
                mail to: authorEmail,
                     subject: "Build Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                     body: "Build successful. Check console: ${env.BUILD_URL}"
                 
                slackSend color: 'good', 
                          channel: '#ariel-jenkins',
                          message: "Build Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}] by ${authorEmail} (<${env.BUILD_URL}|Open>)"
            }
        }
        failure {
            script {
                def authorEmail = sh(returnStdout: true, script: "git show -s --format='%ae'").trim()
                mail to: authorEmail,
                     subject: "Build Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                     body: "Build failed. Check console: ${env.BUILD_URL}"

                slackSend color: 'danger', 
                          channel: '#ariel-jenkins',
                          message: "Build Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}] by ${authorEmail} (<${env.BUILD_URL}|Open>)"
            }
        }
    }
}