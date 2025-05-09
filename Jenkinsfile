pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = "dockerhub-username"
        APP_NAME = "python-app"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint') {
            steps {
                sh 'pip install pylint'
                sh 'pylint --disable=C0111,C0103,C0303,C0330 app.py || true'
            }
        }

        stage('Test') {
            steps {
                sh 'pip install pytest flask'
                sh 'pytest'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    pip install coverage
                    coverage run -m pytest
                    coverage xml
                    sonar-scanner \
                      -Dsonar.projectKey=python-app \
                      -Dsonar.sources=. \
                      -Dsonar.python.coverage.reportPaths=coverage.xml \
                      -Dsonar.host.url=${SONAR_HOST_URL} \
                      -Dsonar.login=${SONAR_AUTH_TOKEN}
                    '''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'docker-hub-password', variable: 'DOCKER_PASSWORD')]) {
                    sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_REGISTRY} --password-stdin"
                    sh "docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "sed -i 's|\\\${DOCKER_REGISTRY}|${DOCKER_REGISTRY}|g; s|\\\${BUILD_NUMBER}|${BUILD_NUMBER}|g' deployment.yaml"
                    sh "kubectl apply -f deployment.yaml"
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} || true"
            cleanWs()
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
