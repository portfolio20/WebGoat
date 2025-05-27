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
                    aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('🚀 Push to ECR') {
            steps {
                sh 'docker push $ECR_REPO:$IMAGE_TAG'
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

        stage('🛡️ DAST Scan - Nikto') {
            steps {
                sshagent(['nikto-ec2-key']) {
                    sh '''
                    echo "[+] Nikto 원격 스캔 시작"
                    ssh -o StrictHostKeyChecking=no ec2-user@<NIKTO_EC2_PUBLIC_IP> '
                      cd ~/nikto/program &&
                      perl nikto.pl -h http://webgoat-alb-1644780168.ap-northeast-2.elb.amazonaws.com \
                        -output webgoat-nikto.html -Format html
                    '

                    echo "[+] Nikto 리포트 가져오는 중..."
                    scp -o StrictHostKeyChecking=no ec2-user@<NIKTO_EC2_PUBLIC_IP>:~/nikto/program/webgoat-nikto.html .

                    echo "[+] S3에 리포트 업로드 중..."
                    aws s3 cp webgoat-nikto.html s3://webgoat-dast-report-bucket/reports/webgoat-nikto-$(date +%Y%m%d-%H%M%S).html --region ap-northeast-2
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'webgoat-nikto.html', allowEmptyArchive: true
                }
            }
        }

        stage('🧼 Nikto 결과 요약 및 알림 처리') {
            steps {
                script {
                    def findings = sh(
                        script: "grep -o '[0-9]* item(s) reported' webgoat-nikto.html | grep -o '^[0-9]*'",
                        returnStdout: true
                    ).trim()

                    echo "🛡️ Nikto 스캔 결과 요약: ${findings}건의 취약 항목이 보고되었습니다."

                    if (findings.toInteger() > 0) {
                        echo "⚠️ 취약점이 발견되어 SNS 알림을 보냅니다."

                        sh '''
                        aws sns publish \
                          --topic-arn arn:aws:sns:ap-northeast-2:669155637873:webgoat-dast-alert-topic \
                          --subject "🚨 Nikto 취약점 경고" \
                          --message "Nikto 스캔 결과 ${findings}건의 취약점이 탐지되었습니다. 리포트를 확인하십시오." \
                          --region ap-northeast-2
                        '''

                        error("❌ Nikto 스캔 결과에 따라 빌드 실패 처리합니다.")
                    } else {
                        echo "✅ 취약점이 발견되지 않았습니다. 문제없습니다."
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Successfully built, pushed, deployed, and scanned!"
        }
        failure {
            echo "❌ Build or deployment failed. Check logs!"
        }
    }
}
