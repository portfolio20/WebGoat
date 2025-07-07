pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        IMAGE_NAME = 'jenkins-demo'
        ACCOUNT_ID = '805369546017'
        ECR_URL = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/webgoat-ecr"
    }

    tools {
        jdk 'jdk-23'
    }

    stages {
        stage('DEBUG JAVA') {
            steps {
                sh '''
                  echo "JAVA_HOME=$JAVA_HOME"
                  java -version
                  mvn -version
                '''
            }
        }

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
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-access-key-id']
                ]) {
                    sh '''
                      aws ecr get-login-password --region $AWS_REGION | \
                      docker login --username AWS --password-stdin $ECR_URL
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                  docker tag $IMAGE_NAME:latest $ECR_URL:latest
                  docker push $ECR_URL:latest
                '''
            }
        }
    }
}
