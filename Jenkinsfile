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
                // Sử dụng đường dẫn đầy đủ đến Python hoặc đảm bảo Python đã có trong PATH
                bat 'python -m pip install pylint'
                bat 'python -m pylint --disable=C0111,C0103,C0303,C0330 app.py || exit 0'
            }
        }
        stage('Test') {
            steps {
                bat 'python -m pip install pytest flask'
                bat 'python -m pytest'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    // Chuyển từ sh sang bat cho Windows
                    bat '''
                    python -m pip install coverage
                    python -m coverage run -m pytest
                    python -m coverage xml
                    '''
                    
                    // Sử dụng SonarScanner cho Windows
                    bat "sonar-scanner.bat -Dsonar.projectKey=python-app -Dsonar.sources=. -Dsonar.python.coverage.reportPaths=coverage.xml -Dsonar.host.url=%SONAR_HOST_URL% -Dsonar.login=%SONAR_AUTH_TOKEN%"
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                // Chuyển từ sh sang bat cho Windows và sửa cú pháp biến môi trường
                bat "docker build -t %DOCKER_REGISTRY%/%APP_NAME%:%BUILD_NUMBER% ."
            }
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'docker-hub-password', variable: 'DOCKER_PASSWORD')]) {
                    // Đối với Windows, cần xử lý đăng nhập Docker khác
                    bat "docker login -u %DOCKER_REGISTRY% -p %DOCKER_PASSWORD%"
                    bat "docker push %DOCKER_REGISTRY%/%APP_NAME%:%BUILD_NUMBER%"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    // Sử dụng PowerShell để thay thế chuỗi trong file
                    powershell '''
                    (Get-Content deployment.yaml) | 
                    ForEach-Object {$_ -replace '\\${DOCKER_REGISTRY}', $env:DOCKER_REGISTRY -replace '\\${BUILD_NUMBER}', $env:BUILD_NUMBER} | 
                    Set-Content deployment.yaml
                    '''
                    bat "kubectl apply -f deployment.yaml"
                }
            }
        }
    }
    post {
        always {
            // Xóa image Docker và làm sạch workspace
            bat "docker rmi %DOCKER_REGISTRY%/%APP_NAME%:%BUILD_NUMBER% || exit 0"
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
