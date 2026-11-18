
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'devops-ecommerce'
        SONAR_PROJECT_KEY = 'devops-ecommerce'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Python Setup') {
            steps {
                echo 'Checking Python version...'
                sh '''
                    python3 --version
                    pip3 --version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Python dependencies...'
                sh '''
                    if [ -f requirements.txt ]; then
                        python3 -m pip install -r requirements.txt --break-system-packages
                    else
                        echo "requirements.txt not found"
                    fi
                '''
            }
        }

        stage('Python Tests') {
            steps {
                echo 'Running Python tests...'
                sh '''
                    if [ -d tests ]; then
                        python3 -m pytest tests/ || exit 1
                    else
                        echo "No tests directory found. Skipping tests."
                    fi
                '''
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=devops-ecommerce \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,.venv/**,**/__pycache__/**,**/tests/** \
                            -Dsonar.sourceEncoding=UTF-8
                        """
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'
                sh '''
                    docker build -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Docker Image Check') {
            steps {
                sh '''
                    docker images | grep devops-ecommerce
                '''
            }
        }
    }

    post {
        success {
            echo 'E-commerce pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}

