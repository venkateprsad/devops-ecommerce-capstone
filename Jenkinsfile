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

        // DEV server
        DEV_HOST = '172.31.45.9'
        DEV_USER = 'ubuntu'
        DEV_CREDENTIALS = 'dev-server-key'

        // STAGE server
        STAGE_HOST = '172.31.34.249'
        STAGE_USER = 'ubuntu'
        STAGE_CREDENTIALS = 'stage-server-key'

        // PRODUCTION server
        PROD_HOST = '172.31.38.11'
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
                        echo "requirements.txt not found. Skipping dependencies."
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
                echo 'Deploying to DEV environment...'

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
                echo 'Deploying to STAGE environment...'

                sshagent(credentials: [env.STAGE_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                          $STAGE_USER@$STAGE_HOST \
                          "set -e;
                           aws ecr get-login-password --region $AWS_REGION |
                           docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com;
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
                        message: 'DEV and STAGE deployments successful. Approve Production deployment?',
                        ok: 'Approve Production',
                        submitter: 'admin'
                    )
                }
            }
        }

        stage('Deploy to PRODUCTION') {
            steps {
                echo 'Deploying to PRODUCTION environment...'

                sshagent(credentials: [env.PROD_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                          $PROD_USER@$PROD_HOST \
                          "set -e;
                           aws ecr get-login-password --region $AWS_REGION |
                           docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com;
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

                    <p>Your Ecommerce application has completed the deployment pipeline.</p>

                    <h3>Deployment Summary</h3>

                    <ul>
                        <li>GitHub Checkout: SUCCESS</li>
                        <li>Python Setup: SUCCESS</li>
                        <li>Python Dependencies: SUCCESS or SKIPPED</li>
                        <li>Python Tests: SUCCESS or SKIPPED</li>
                        <li>SonarQube Analysis: SUCCESS</li>
                        <li>Docker Build: SUCCESS</li>
                        <li>AWS ECR Push: SUCCESS</li>
                        <li>DEV Deployment: SUCCESS</li>
                        <li>STAGE Deployment: SUCCESS</li>
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

                    <p>Your Ecommerce CI/CD pipeline has failed.</p>

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
            echo 'Pipeline was aborted.'

            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "ABORTED: Ecommerce CI/CD Build #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2>CI/CD Pipeline Aborted</h2>
                    <p>The pipeline was aborted or production approval timed out.</p>
                    <p><b>Build:</b> ${BUILD_NUMBER}</p>
                    <p><a href="${BUILD_URL}">View Jenkins Build</a></p>
                """
            )
        }
        

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
