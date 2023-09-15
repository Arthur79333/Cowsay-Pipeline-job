pipeline {
    agent any

    triggers {
        pollSCM('H/5 * * * *')  // polls every 5 minutes
    }

    options {
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        COWSAY_IMG="cowsay"
        COWSAY_TEST="cowsay-test"
	COWSAY_PORT=8080
        ECR_REPO="644435390668.dkr.ecr.il-central-1.amazonaws.com/ec2-arthur"
    }

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t ${COWSAY_IMG} .'
            }
        }  

        stage('Test') {
            steps {
                sh """
                    docker run --name ${COWSAY_TEST} --network="jenkins_default" -d ${COWSAY_IMG}
                    sleep 5
                    curl -fsSLi "http://${COWSAY_TEST}:${COWSAY_PORT}" --max-time 1
                    docker rm -f ${COWSAY_TEST}
                """
            }
        }  

        stage('Tag') {
            steps {
                sh '''
                    docker tag ${COWSAY_IMG}:latest ${ECR_REPO}:${BUILD_NUMBER}
                    docker rmi ${COWSAY_IMG}:latest
                '''
            }
        }
        
        stage('Push') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'aws_credentials']]) {
                    // check maybe need to add path to    /usr/bin/aws 
                    sh '''
                        /usr/bin/aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        /usr/bin/aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        /usr/bin/aws configure set region il-central-1
                        LOGIN_PASSWORD=$(/usr/bin/aws ecr get-login-password --region il-central-1)
                        docker login -u AWS -p \${LOGIN_PASSWORD} ${ECR_REPO}                        
                        docker push ${ECR_REPO}:${BUILD_NUMBER}
                    '''
                }
            }
        }        
        
        stage('Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2_jenkins', keyFileVariable: 'SSH_KEY')
                ]) {
                    sh """
                        # SSH into the EC2 instance and run the commands.
                        ssh -i \$SSH_KEY admin@51.16.152.185 "
                            echo "Connected to EC2"                            
                            echo "Logging into ECR through Docker"
                            docker login -u AWS -p \$(/usr/bin/aws ecr get-login-password --region il-central-1) \${ECR_REPO}
                            echo "Login successful"                            
                            docker pull \${ECR_REPO}:\${BUILD_NUMBER}                            
                            docker run -d -p \${COWSAY_PORT}:\${COWSAY_PORT} --name \${COWSAY_IMG} \${ECR_REPO}:\${BUILD_NUMBER}
                        "
                    """
                }
            }
        }
    }    

    post {
        always {
            sh '''
                #docker rmi cowsay
                #docker rmi "${ECR_REPO}:${BUILD_NUMBER}"
                #docker rmi "${ECR_REPO}:latest"
            '''
            cleanWs()
        }  

        failure {
            updateGitlabCommitStatus name: 'Jenkins', state: 'failed'
        }

        success {
            updateGitlabCommitStatus name: 'Jenkins', state: 'success'
        }
    }
}
