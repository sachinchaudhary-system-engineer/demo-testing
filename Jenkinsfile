pipeline {
    agent any

    environment {
        APP_NAME  = "my-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        USER_DOCKER = "chaudharysachin"
        SONAR_SCANNER = tool 'SonarScanner'
    }

    stages {

        stage("Checkout") {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sachinchaudhary-system-engineer/demo-testing.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube') {
                   sh '''
                ${SONAR_SCANNER}/bin/sonar-scanner
                  '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh '''
                    docker build -t ${APP_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage("Push to Docker Hub") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login \
                            -u "${DOCKER_USER}" \
                            --password-stdin

                        docker tag ${APP_NAME}:${IMAGE_TAG} \
                            ${USER_DOCKER}/${APP_NAME}:${IMAGE_TAG}

                        docker push \
                            ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Manual Approval') {
            steps {
                input message: 'Approve this image for Docker Hub?', 
                      ok: 'Approve'
            }
        }

        stage("Deploy") {
            steps {
                sh '''
                    kubectl apply -f k8s/namespace-1.yaml
                    kubectl apply -f k8s/

                    kubectl set image deployment/demo-deployment \
                        demo-container=${USER_DOCKER}/${APP_NAME}:${IMAGE_TAG} -n jenkins

                    kubectl rollout restart deployment/demo-deployment -n jenkins
                    kubectl rollout status deployment/demo-deployment -n jenkins
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment successful"
        }

        failure {
            echo "Deployment failed"
        }

        always {
            sh 'docker logout || true'
        }
    }
}
