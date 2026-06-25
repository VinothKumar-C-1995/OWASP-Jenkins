pipeline {

    agent any

    tools {
        maven 'Maven'
    }

    environment {
        IMAGE_NAME = "vinothkumaraws/owasp-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        APP_SERVER = 16.170.133.87
        DEPENDENCY_CHECK = "/opt/dependency-check/bin/dependency-check.sh"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn sonar:sonar
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {

                withCredentials([
                    string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')
                ]) {

                    sh '''
                    ${DEPENDENCY_CHECK} \
                    --project "OWASP-Jenkins" \
                    --scan . \
                    --out dependency-check-report \
                    --format XML \
                    --format HTML \
                    --nvdApiKey $NVD_API_KEY
                    '''
                }
            }
        }

        stage('Publish OWASP Report') {
            steps {
                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report/dependency-check-report.xml'
                )
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to EC2') {

            steps {

                sshagent(['ec2-key']) {

                    sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@${APP_SERVER} "

                    docker pull ${IMAGE_NAME}:latest

                    docker stop java-app || true

                    docker rm java-app || true

                    docker run -d \
                    --name java-app \
                    -p 8080:8080 \
                    ${IMAGE_NAME}:latest

                    "
                    '''
                }

            }

        }

    }

    post {

        success {

            echo '================================='
            echo 'Pipeline completed successfully!'
            echo '================================='

        }

        failure {

            echo '================================='
            echo 'Pipeline failed!'
            echo '================================='

        }

        always {

            archiveArtifacts artifacts: 'dependency-check-report/**/*', fingerprint: true

            cleanWs()

        }

    }

}
