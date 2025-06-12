pipeline {
    agent any
    
    tools {
        maven "M3"
        jdk "JDK17"
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('ksy-docker')
        REGION = "ap-northeast-2"
        AWS_CREDENTIALS_NAME = "project4-aws-credentials"
    }
    
    stages {
        stage('Git Clone') {
            steps {
                echo 'Git Clone'
                git url: 'https://github.com/yytjdduq/spring-petclinic.git',
                    branch: 'aws-branch'
            }
            post {
                success {
                    echo 'Git Clone Success'
                }
                failure {
                    echo 'Git Clone Fail'
                }
            }
        }
        // Maven Build 작업
        stage('Maven Build') {
            steps {
                echo 'Maven Build'
                sh 'mvn -Dmaven.test.failure.ignore=true clean package' // Test error 무시
            }
        }

        // Docker Image 생성
        stage('Docker Image Build') {
            steps {
                echo 'Docker Image Build'
                dir("${env.WORKSPACE}") {
                    sh '''
                       docker build -t spring-petclinic:$BUILD_NUMBER .
                       docker tag spring-petclinic:$BUILD_NUMBER ksy99/spring-petclinic:latest
                       '''
                }
            }
        }
        
        // Docker Image Push
        stage('Docker Image Push') {
            steps {
                sh '''
                   echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                   docker push ksy99/spring-petclinic:latest
                   '''
            }
        }
        // Remover Docker Image
        stage('Remove Docker Image') {
            steps {
                sh '''
                docker rmi spring-petclinic:$BUILD_NUMBER
                docker rmi ksy99/spring-petclinic:latest
                '''
            }
        }
        stage('Upload S3') {
            steps {
                echo "Upload to S3"
                dir("${env.WORKSPACE}") {
                    sh 'zip -r scripts.zip ./scripts appspec.yml'
                    withAWS(region:"${REGION}", credentials:"${AWS_CREDENTIALS_NAME}"){
                        s3Upload(file:"scripts.zip", bucket:"project4-bucket-tg")
                    }
                    sh 'rm -rf ./scripts.zip' 
                }
            }
        }
        stage('Codedeploy Workload') {
            steps {
                echo "Codedeploy Workload"   
                withAWS(region:"${REGION}", credentials:"${AWS_CREDENTIALS_NAME}"){
                    sh '''
                    aws deploy create-deployment --application-name project4-application \
                    --deployment-config-name CodeDeployDefault.OneAtATime \
                    --deployment-group-name project4-production-in_place \
                    --ignore-application-stop-failures \
                    --s3-location bucket=project4-bucket-tg,bundleType=zip,key=scripts.zip
                    '''
                }
                sleep(10) // sleep 10s
            }
        }
    }
}
