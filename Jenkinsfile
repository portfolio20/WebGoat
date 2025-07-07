pipeline {
    agent any

    environment {
        AWS_REGION  = 'ap-northeast-2'
        ACCOUNT_ID  = '805369546017'
        IMAGE_NAME  = 'jenkins-demo'
        ECR_URL     = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/webgoat-ecr"
        S3_BUCKET   = 'jenkins-webgoat-bucket'
        BUNDLE      = 'webgoat-deploy-bundle.zip'
        DEPLOY_APP  = 'webgoat-cd-app'
        DEPLOY_GROUP = 'webgoat-deployment-group'
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

        stage('Upload to S3') {
            steps {
                sh '''
                    zip -r $BUNDLE appspec.yml imagedefinitions.json
                    aws s3 cp $BUNDLE s3://$S3_BUCKET/
                '''
            }
        }

        stage('Deploy to ECS (CodeDeploy)') {
            steps {
                sh '''
                    aws deploy create-deployment \
                      --application-name $DEPLOY_APP \
                      --deployment-group-name $DEPLOY_GROUP \
                      --s3-location bucket=$S3_BUCKET,bundleType=zip,key=$BUNDLE \
                      --region $AWS_REGION
                '''
            }
        }
    }
}
