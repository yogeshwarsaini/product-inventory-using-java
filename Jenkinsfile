pipeline {
    agent any

    tools {
        maven 'Maven3'   // Jenkins Tools mein jo naam diya tha
    }

    environment {
        // ✏️ Ye 5 values apni actual values se badal do 
        S3_BUCKET     = 'cdk-hnb659fds-assets-678804053714-ap-south-1'
        EB_APP_NAME   = 'my-java-app'
        EB_ENV_NAME   = 'My-java-app-env'
        AWS_REGION    = 'ap-south-1'

        // Ye mat badlo — automatically set hoti hain
        ARTIFACT_NAME = "app-${env.BUILD_NUMBER}.zip"
        JAR_PATH      = "target/*.jar"
    }

    stages {

        // ─────────────────────────────────────────
        // STAGE 1: Code GitHub se fetch karlo
	
        // ─────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Code checkout ho gaya — Branch: ${env.GIT_BRANCH}"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 2: Maven se JAR build karo
        // ─────────────────────────────────────────
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                echo "✅ Build ho gaya"
            }
            post {
                success {
                    // Build artifacts archive karo (Jenkins UI mein dikhenge)
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        // ─────────────────────────────────────────
        // STAGE 3: JUnit tests chalao
        // ─────────────────────────────────────────
        // stage('Test') {
        //     steps {
        //         sh 'mvn test'
        //         echo "✅ Tests pass ho gaye"
        //     }
        //     post {
        //         always {
        //             // Test results Jenkins dashboard pe dikhenge
        //             junit '**/target/surefire-reports/*.xml'
        //         }
        //     }
        // }

        // ─────────────────────────────────────────
        // STAGE 4: JAR ko ZIP mein pack karo
        // EB ko ZIP format chahiye hota hai
        // ─────────────────────────────────────────
        stage('Package') {
            steps {
                sh '''
                    # Pehli baar ke liye deploy folder clean karo
                    rm -rf deploy && mkdir deploy

                    # JAR copy karo deploy folder mein
                    cp target/*.jar deploy/app.jar

                    # Procfile banao — EB isse Java app start karta hai
                    echo "web: java -jar app.jar" > deploy/Procfile

                    # ZIP banao
                    cd deploy && zip -r ../${ARTIFACT_NAME} .
                '''
                echo "✅ ${ARTIFACT_NAME} ready hai"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 5: ZIP ko S3 pe upload karo
        // ─────────────────────────────────────────
        stage('Upload to S3') {
            steps {
                sh """
                    aws s3 cp ${ARTIFACT_NAME} \
                        s3://${S3_BUCKET}/builds/${ARTIFACT_NAME} \
                        --region ${AWS_REGION}
                """
                echo "✅ S3 pe upload ho gaya: builds/${ARTIFACT_NAME}"
            }
        }

        // ─────────────────────────────────────────
        // STAGE 6: Elastic Beanstalk pe deploy karo
        // ─────────────────────────────────────────
        stage('Deploy to Elastic Beanstalk') {
            steps {
                sh """
                    # Nayi application version register karo
                    aws elasticbeanstalk create-application-version \
                        --application-name ${EB_APP_NAME} \
                        --version-label build-${env.BUILD_NUMBER} \
                        --source-bundle S3Bucket=${S3_BUCKET},S3Key=builds/${ARTIFACT_NAME} \
                        --region ${AWS_REGION}

                    # Environment mein deploy karo
                    aws elasticbeanstalk update-environment \
                        --application-name ${EB_APP_NAME} \
                        --environment-name ${EB_ENV_NAME} \
                        --version-label build-${env.BUILD_NUMBER} \
                        --region ${AWS_REGION}

                    # Deploy complete hone ka wait karo (max 5 min)
                    aws elasticbeanstalk wait environment-updated \
                        --application-name ${EB_APP_NAME} \
                        --environment-name ${EB_ENV_NAME} \
                        --region ${AWS_REGION}
                """
                echo "✅ Elastic Beanstalk pe deploy ho gaya!"
            }
        }
    }

    // ─────────────────────────────────────────
    // Pipeline ke baad cleanup + notification
    // ─────────────────────────────────────────
    post {
        success {
            echo "🎉 Pipeline SUCCESS — Build #${env.BUILD_NUMBER} live hai!"
        }
        failure {
            echo "💥 Pipeline FAILED — Console Output check karo"
        }
        always {
            // Workspace clean karo (disk space bachao)
            cleanWs()
        }
    }
}
