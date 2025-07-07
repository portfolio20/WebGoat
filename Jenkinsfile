pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        IMAGE_NAME = 'jenkins-demo'
        ACCOUNT_ID = '805369546017'
        ECR_URL = "805369546017.dkr.ecr.ap-northeast-2.amazonaws.com/webgoat-ecr"
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
                    sh """
                        echo "Logging into ECR with region: ${env.AWS_REGION}"
                        aws ecr get-login-password --region ${env.AWS_REGION} | \
                        docker login --username AWS --password-stdin ${env.ECR_URL}
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    docker tag ${env.IMAGE_NAME}:latest ${env.ECR_URL}/${env.IMAGE_NAME}:latest
                    docker push ${env.ECR_URL}/${env.IMAGE_NAME}:latest
                """
            }
        }
    }
}
