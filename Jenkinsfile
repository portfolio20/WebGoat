pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        IMAGE_NAME = 'jenkins-demo'
        ACCOUNT_ID = '805369546017'
        ECR_URL = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('ECR Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_URL
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                docker tag $IMAGE_NAME:latest $ECR_URL/$IMAGE_NAME:latest
                docker push $ECR_URL/$IMAGE_NAME:latest
                '''
            }
        }
    }
}
