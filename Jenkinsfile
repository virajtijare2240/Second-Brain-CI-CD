pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'PUSH_TO_DOCKERHUB', defaultValue: false,
                     description: 'Push the built images to Docker Hub (requires Jenkins credential dockerhub-creds).')
    }

    environment {
        BACKEND_IMAGE = 'second-brain-backend'
        FRONTEND_IMAGE = 'second-brain-frontend'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code is checked out from the configured GitHub repository.'
                checkout scm
            }
        }

        stage('Backend Setup and Tests') {
            steps {
                dir('backend') {
                    bat '"C:\\Users\\viraj\\AppData\\Local\\Programs\\Python\\Python313\\python.exe" -m venv .venv-ci'
                    bat '.venv-ci\\Scripts\\python.exe -m pip install --upgrade pip'
                    bat '.venv-ci\\Scripts\\python.exe -m pip install -r requirements.txt'
                    bat '.venv-ci\\Scripts\\python.exe -m pytest -v --junitxml=junit-results.xml'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/**/junit*.xml'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    bat 'npm.cmd ci'
                    bat 'npm.cmd run build'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                bat 'docker build -t %BACKEND_IMAGE%:%IMAGE_TAG% -t %BACKEND_IMAGE%:latest ./backend'
                bat 'docker build -t %FRONTEND_IMAGE%:%IMAGE_TAG% -t %FRONTEND_IMAGE%:latest ./frontend'
            }
        }

        stage('Push Images to Docker Hub') {
            when {
                expression { return params.PUSH_TO_DOCKERHUB }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_TOKEN')]) {
                    bat 'echo %DOCKERHUB_TOKEN% | docker login -u %DOCKERHUB_USER% --password-stdin'
                    bat 'docker tag %BACKEND_IMAGE%:%IMAGE_TAG% %DOCKERHUB_USER%/%BACKEND_IMAGE%:%IMAGE_TAG%'
                    bat 'docker tag %FRONTEND_IMAGE%:%IMAGE_TAG% %DOCKERHUB_USER%/%FRONTEND_IMAGE%:%IMAGE_TAG%'
                    bat 'docker push %DOCKERHUB_USER%/%BACKEND_IMAGE%:%IMAGE_TAG%'
                    bat 'docker push %DOCKERHUB_USER%/%FRONTEND_IMAGE%:%IMAGE_TAG%'
                    bat 'docker logout'
                }
            }
        }

        stage('Deploy Locally') {
            steps {
                bat 'docker compose up -d --build'
            }
        }
    }

    post {
        success {
            echo 'Second Brain CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Open the failed stage console output, fix the reported issue, and run the build again.'
        }
    }
}
