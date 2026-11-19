
pipeline {
    agent any

    environment {
        // Application
        IMAGE_NAME = 'devops-ecommerce'
        SONAR_PROJECT_KEY = 'devops-ecommerce'
        CONTAINER_NAME = 'ecommerce'

        // AWS ECR
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '270278216994'
        ECR_URI = '270278216994.dkr.ecr.ap-southeast-2.amazonaws.com/devops-ecommerce'

        // Stage server
        STAGE_HOST = '16.178.133.152'
        STAGE_USER = 'ubuntu'
        STAGE_CREDENTIALS = 'stage-server-key'

        // Dev server - fill actual details
        DEV_HOST = 'REPLACE_WITH_DEV_PRIVATE_IP'
        DEV_USER = 'ubuntu'
        DEV_CREDENTIALS = 'dev-server-key'

        // Production server - fill actual details
        PROD_HOST = 'REPLACE_WITH_PRODUCTION_PRIVATE_IP'
        PROD_USER = 'ubuntu'
        PROD_CREDENTIALS = 'prod-server-key'

        // Email
        NOTIFY_EMAIL = 'gochhivenkata@gmail.com'
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
                script {
                    env.IMAGE_TAG = env.BUILD_NUMBER
                }

                echo 'Building Docker image...'

                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${IMAGE_NAME}:latest .
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

        stage('AWS ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region $AWS_REGION | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    docker tag \
                      $IMAGE_NAME:$IMAGE_TAG \
                      $ECR_URI:$IMAGE_TAG

                    docker tag \
                      $IMAGE_NAME:latest \
                      $ECR_URI:latest

                    docker push $ECR_URI:$IMAGE_TAG
                    docker push $ECR_URI:latest
                '''
            }
        }

        stage('Deploy to DEV') {
            steps {
                echo 'Deploying to Dev environment...'

                sshagent(credentials: [env.DEV_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                          $DEV_USER@$DEV_HOST \
                          "set -e;
                           aws ecr get-login-password --region $AWS_REGION |
                           docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com;
                           docker pull $ECR_URI:$IMAGE_TAG;
                           docker rm -f $CONTAINER_NAME || true;
                           docker run -d --restart unless-stopped --name $CONTAINER_NAME -p 80:80 $ECR_URI:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy to STAGE') {
            steps {
                echo 'Deploying to Stage environment...'

                sshagent(credentials: [env.STAGE_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                          $STAGE_USER@$STAGE_HOST \
                          "set -e;
                           aws ecr get-login-password --region $AWS_REGION |
                           docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com;
                           docker pull $ECR_URI:$IMAGE_TAG;
                           docker rm -f $CONTAINER_NAME || true;
                           docker run -d --restart unless-stopped --name $CONTAINER_NAME -p 80:80 $ECR_URI:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Production Approval') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: 'Dev and Stage deployments successful. Approve Production deployment?',
                        ok: 'Approve Production',
                        submitter: 'admin'
                    )
                }
            }
        }

        stage('Deploy to PRODUCTION') {
            steps {
                echo 'Deploying to Production environment...'

                sshagent(credentials: [env.PROD_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                          $PROD_USER@$PROD_HOST \
                          "set -e;
                           aws ecr get-login-password --region $AWS_REGION |
                           docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com;
                           docker pull $ECR_URI:$IMAGE_TAG;
                           docker rm -f $CONTAINER_NAME || true;
                           docker run -d --restart unless-stopped --name $CONTAINER_NAME -p 80:80 $ECR_URI:$IMAGE_TAG"
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'E-commerce CI/CD pipeline completed successfully!'

            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "SUCCESS: Ecommerce CI/CD Build #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:green;">CI/CD Pipeline Successful!</h2>

                    <p>Hello,</p>

                    <p>Your Ecommerce application has been successfully deployed.</p>

                    <h3>Deployment Summary</h3>

                    <ul>
                        <li>GitHub Checkout: SUCCESS</li>
                        <li>Python Dependencies: SUCCESS</li>
                        <li>Python Tests: SUCCESS</li>
                        <li>SonarQube Analysis: SUCCESS</li>
                        <li>Docker Build: SUCCESS</li>
                        <li>AWS ECR Push: SUCCESS</li>
                        <li>Dev Deployment: SUCCESS</li>
                        <li>Stage Deployment: SUCCESS</li>
                        <li>Production Approval: GRANTED</li>
                        <li>Production Deployment: SUCCESS</li>
                    </ul>

                    <p><b>Project:</b> ${JOB_NAME}</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Docker Image:</b> ${ECR_URI}:${IMAGE_TAG}</p>
                    <p><b>Build URL:</b> <a href="${BUILD_URL}">${BUILD_URL}</a></p>

                    <p>Regards,<br>DevOps CI/CD Pipeline</p>
                """
            )
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'

            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "FAILED: Ecommerce CI/CD Build #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:red;">CI/CD Pipeline Failed!</h2>

                    <p>Your Ecommerce deployment pipeline has failed.</p>

                    <p><b>Project:</b> ${JOB_NAME}</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Status:</b> FAILURE</p>

                    <p>Please check Jenkins console output:</p>
                    <a href="${BUILD_URL}console">${BUILD_URL}console</a>

                    <p>Regards,<br>DevOps CI/CD Pipeline</p>
                """
            )
        }

        aborted {
            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "ABORTED: Ecommerce CI/CD Build #${BUILD_NUMBER}",
                body: "The pipeline was aborted or the approval timed out. Check ${BUILD_URL}"
            )
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
