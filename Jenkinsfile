pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/venkateprsad/devops-ecommerce-capstone.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ecommerce-frontend:latest .'
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker stop ecommerce-app || true
                    docker rm ecommerce-app || true
                    docker run -d \
                    --name ecommerce-app \
                    -p 8080:80 \
                    ecommerce-frontend:latest
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'sleep 5'
                sh 'curl -f http://localhost:8080'
            }
        }
    }
}
