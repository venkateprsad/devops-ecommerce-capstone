
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub'
            }
        }

        stage('Build and Deploy') {
            steps {
                sh '''
                    docker compose -p devops-ecommerce-capstone up -d --build
                '''
            }
        }

        stage('Verify Containers') {
            steps {
                sh '''
                    docker compose -p devops-ecommerce-capstone ps
                    sleep 5
                    curl -f http://localhost
                '''
            }
        }
    }

    post {
        success {
            echo 'E-commerce application deployed successfully!'
        }

        failure {
            echo 'Deployment failed. Check Jenkins console output.'
        }
    }
}

