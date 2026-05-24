pipeline {
    agent any

    environment {
        S3_BUCKET      = 'cdk-hnb659fds-assets-678804053714-ap-south-1'
        EB_APP_NAME    = 'my-java-app'
        EB_ENV_NAME    = 'my-java-app-env'
        AWS_REGION     = 'ap-south-1'
        ARTIFACT_NAME  = "app-${BUILD_NUMBER}.zip"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                // Gradle use karte ho toh: sh './gradlew build -x test'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh '''
                    mkdir -p deploy
                    cp target/*.jar deploy/
                    cd deploy && zip -r ../${ARTIFACT_NAME} .
                '''
            }
        }

        stage('Upload to S3') {
            steps {
                sh """
                    aws s3 cp ${ARTIFACT_NAME} s3://${S3_BUCKET}/${ARTIFACT_NAME} \
                        --region ${AWS_REGION}
                """
            }
        }

        stage('Deploy to Elastic Beanstalk') {
            steps {
                sh """
                    aws elasticbeanstalk create-application-version \
                        --application-name ${EB_APP_NAME} \
                        --version-label ${BUILD_NUMBER} \
                        --source-bundle S3Bucket=${S3_BUCKET},S3Key=${ARTIFACT_NAME} \
                        --region ${AWS_REGION}

                    aws elasticbeanstalk update-environment \
                        --application-name ${EB_APP_NAME} \
                        --environment-name ${EB_ENV_NAME} \
                        --version-label ${BUILD_NUMBER} \
                        --region ${AWS_REGION}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deploy successful! Build #${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Pipeline failed at some stage"
        }
    }
}
