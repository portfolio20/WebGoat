pipeline {
    agent any

    environment {
        ECR_REPO = "669155637873.dkr.ecr.ap-northeast-2.amazonaws.com/jenkins-demo"
        IMAGE_TAG = "latest"
        JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        S3_BUCKET = "webgoat-codedeploy-bucket"
        DEPLOY_APP = "jenkins-app"
        DEPLOY_GROUP = "jenkins-deploy-group"
        REGION = "ap-northeast-2"
        BUNDLE = "webgoat-deploy-bundle.zip"
    }

    stages {
        stage('📦 Checkout') {
            steps {
                checkout scm
            }
        }

        stage('🔨 Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('🐳 Docker Build') {
            steps {
                sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
            }
        }

        stage('🔐 ECR Login') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${REGION}") {
                    sh '''
                    aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('🚀 Push to ECR') {
            steps {
                sh 'docker push $ECR_REPO:$IMAGE_TAG'
            }
        }

        stage('🧪 Run ZAP Scan via SSH') {
            steps {
                sshagent(credentials: ['nikto-private-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@<ZAP_IP> '
                        aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REPO &&
                        docker rm -f webgoat-test || true &&
                        docker run -d --name webgoat-test -p 8080:8080 $ECR_REPO:$IMAGE_TAG &&
                        /opt/zaproxy/scan.sh webgoat-test 8090 /start.mvc
                    '
                    """
                }
            }
        }
        

        stage('🧩 Generate taskdef.json') {
            steps {
                script {
                    def taskdef = """{
  \"family\": \"webgoat-taskdef\",
  \"networkMode\": \"awsvpc\",
  \"containerDefinitions\": [
    {
      \"name\": \"webgoat\",
      \"image\": \"${ECR_REPO}:${IMAGE_TAG}\",
      \"memory\": 3072,
      \"cpu\": 1024,
      \"essential\": true,
      \"portMappings\": [
        {
          \"containerPort\": 8080,
          \"protocol\": \"tcp\"
        }
      ]
    }
  ],
  \"requiresCompatibilities\": [\"FARGATE\"],
  \"cpu\": \"1024\",
  \"memory\": \"3072\",
  \"executionRoleArn\": \"arn:aws:iam::669155637873:role/ecsTaskExecutionRole\"
}"""
                    writeFile file: 'taskdef.json', text: taskdef
                }
            }
        }

        stage('📄 Generate appspec.yaml') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: "${REGION}") {
                        def taskDefArn = sh(
                            script: "aws ecs register-task-definition --cli-input-json file://taskdef.json --query 'taskDefinition.taskDefinitionArn' --output text",
                            returnStdout: true
                        ).trim()

                        def appspec = """version: 1
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: \"${taskDefArn}\"
        LoadBalancerInfo:
          ContainerName: \"webgoat\"
          ContainerPort: 8080
"""
                        writeFile file: 'appspec.yaml', text: appspec
                    }
                }
            }
        }

        stage('📦 Bundle for CodeDeploy') {
            steps {
                sh 'zip -r $BUNDLE appspec.yaml Dockerfile taskdef.json'
            }
        }

        stage('🚀 Deploy via CodeDeploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: "${REGION}") {
                        sh '''
                        aws s3 cp $BUNDLE s3://$S3_BUCKET/$BUNDLE --region $REGION

                        aws deploy create-deployment \
                          --application-name $DEPLOY_APP \
                          --deployment-group-name $DEPLOY_GROUP \
                          --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
                          --s3-location bucket=$S3_BUCKET,bundleType=zip,key=$BUNDLE \
                          --region $REGION
                        '''
                    }
                }
            }
        }

        stage('🧪 ECR 로그인 및 Pull 테스트') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${REGION}") {
                    sh '''
                    echo "[🔐] ECR 로그인 시도 중..."
                    aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REPO

                    echo "[📦] ECR 이미지 Pull 시도 중..."
                    docker pull $ECR_REPO:$IMAGE_TAG
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Successfully built, pushed, and deployed!"
        }
        failure {
            echo "❌ Build, push, or deployment failed. Check logs!"
        }
    }
}
