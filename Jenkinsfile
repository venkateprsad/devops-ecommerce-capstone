
def deployWithRollback(String targetHost, String targetUser,
                       String credentialId, String environmentName) {
    withEnv([
        "TARGET_HOST=${targetHost}",
        "TARGET_USER=${targetUser}",
        "DEPLOY_ENV=${environmentName}"
    ]) {
        sshagent(credentials: [credentialId]) {
            sh '''
                set -eu

                echo "Deploying to $DEPLOY_ENV..."

                ssh -o StrictHostKeyChecking=no \
                    "$TARGET_USER@$TARGET_HOST" \
                    "bash -s -- '$ECR_URI:$IMAGE_TAG' '$CONTAINER_NAME' '$AWS_REGION' '$AWS_ACCOUNT_ID' '$DEPLOY_ENV'" <<'REMOTE'
set -eu

NEW_IMAGE="$1"
CONTAINER_NAME="$2"
AWS_REGION="$3"
AWS_ACCOUNT_ID="$4"
ENVIRONMENT="$5"

ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

echo "Environment: $ENVIRONMENT"
echo "New image: $NEW_IMAGE"

# Remember the currently running image
OLD_IMAGE=$(docker inspect \
    --format='{{.Config.Image}}' \
    "$CONTAINER_NAME" 2>/dev/null || true)

echo "Previous image: ${OLD_IMAGE:-NONE}"

# Login to AWS ECR
aws ecr get-login-password --region "$AWS_REGION" |
    docker login --username AWS --password-stdin "$ECR_REGISTRY"

# Download the new image
docker pull "$NEW_IMAGE"

# Download the previous image too, if needed for rollback
if [ -n "$OLD_IMAGE" ]; then
    docker pull "$OLD_IMAGE" || {
        echo "WARNING: Previous image could not be downloaded."
        echo "Rollback may not be possible."
    }
fi

# Rollback function
rollback() {
    echo "=================================="
    echo "DEPLOYMENT FAILED - ROLLBACK"
    echo "=================================="

    docker rm -f "$CONTAINER_NAME" 2>/dev/null || true

    if [ -n "$OLD_IMAGE" ] && docker image inspect "$OLD_IMAGE" >/dev/null 2>&1; then
        echo "Restoring previous image: $OLD_IMAGE"

        if docker run -d \
            --restart unless-stopped \
            --name "$CONTAINER_NAME" \
            -p 80:80 \
            "$OLD_IMAGE"; then

            echo "Checking previous application..."

            for attempt in $(seq 1 12); do
                if curl -fsS --max-time 5 http://localhost/ >/dev/null; then
                    echo "Previous version restored and healthy."
                    return 0
                fi
                sleep 5
            done

            echo "WARNING: Previous container started but health check failed."
            return 1
        fi
    fi

    echo "No usable previous image available."
    return 1
}

# Stop old container and deploy new container
docker rm -f "$CONTAINER_NAME" 2>/dev/null || true

if ! docker run -d \
    --restart unless-stopped \
    --name "$CONTAINER_NAME" \
    -p 80:80 \
    "$NEW_IMAGE"; then

    rollback || true
    exit 1
fi

# Application health check
HEALTHY=0

for attempt in $(seq 1 12); do
    echo "Health check attempt $attempt of 12"

    if curl -fsS --max-time 5 http://localhost/ >/dev/null; then
        HEALTHY=1
        break
    fi

    sleep 5
done

# Deployment result
if [ "$HEALTHY" -eq 1 ]; then
    echo "$ENVIRONMENT deployment successful!"
else
    echo "$ENVIRONMENT deployment failed!"

    rollback || true
    exit 1
fi

REMOTE
            '''
        }
    }
}

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

        // Servers
        DEV_HOST = '172.31.45.9'
        DEV_USER = 'ubuntu'
        DEV_CREDENTIALS = 'dev-server-key'

        STAGE_HOST = '172.31.34.249'
        STAGE_USER = 'ubuntu'
        STAGE_CREDENTIALS = 'stage-server-key'

        PROD_HOST = '172.31.38.11'
        PROD_USER = 'ubuntu'
        PROD_CREDENTIALS = 'prod-server-key'

        // Email
        NOTIFY_EMAIL = 'gochhivenkata@gmail.com'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Python Setup') {
            steps {
                sh '''
                    python3 --version
                    python3 -m venv venv
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    . venv/bin/activate
                    pip install --upgrade pip
                    if [ -f requirements.txt ]; then
                        pip install -r requirements.txt
                    fi
                '''
            }
        }

        stage('Python Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    if [ -d tests ]; then
                        python -m pytest tests/ -v
                    else
                        echo "No tests directory found. Skipping tests."
                    fi
                '''
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=venv/**,**/__pycache__/**,.git/**
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}"
                }

                sh '''
                    docker build \
                        -t $IMAGE_NAME:$IMAGE_TAG \
                        -t $IMAGE_NAME:latest .
                '''
            }
        }

        stage('Docker Image Check') {
            steps {
                sh '''
                    docker images | grep "$IMAGE_NAME"
                '''
            }
        }

        stage('AWS ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password --region "$AWS_REGION" |
                    docker login \
                        --username AWS \
                        --password-stdin \
                        "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
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
                        $IMAGE_NAME:$IMAGE_TAG \
                        $ECR_URI:latest

                    docker push $ECR_URI:$IMAGE_TAG
                    docker push $ECR_URI:latest
                '''
            }
        }

        // DEV deployment with automatic rollback
        stage('Deploy to DEV') {
            steps {
                script {
                    deployWithRollback(
                        env.DEV_HOST,
                        env.DEV_USER,
                        env.DEV_CREDENTIALS,
                        'DEV'
                    )
                }
            }
        }

        // STAGE deployment with automatic rollback
        stage('Deploy to STAGE') {
            steps {
                script {
                    deployWithRollback(
                        env.STAGE_HOST,
                        env.STAGE_USER,
                        env.STAGE_CREDENTIALS,
                        'STAGE'
                    )
                }
            }
        }

        // Production approval
        stage('Production Approval') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'DEV and STAGE deployments completed. Approve production deployment?',
                          ok: 'Deploy to Production'
                }
            }
        }

        // PROD deployment with automatic rollback
        stage('Deploy to PRODUCTION') {
            steps {
                script {
                    deployWithRollback(
                        env.PROD_HOST,
                        env.PROD_USER,
                        env.PROD_CREDENTIALS,
                        'PRODUCTION'
                    )
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: 'gochhivenkata@gmail.com',
                subject: "SUCCESS: E-commerce CI/CD Build #${BUILD_NUMBER}",
                body: """
Hello,

Your E-commerce CI/CD pipeline completed successfully.

Build Number: ${BUILD_NUMBER}
Build URL: ${BUILD_URL}

Deployment:
DEV: Successful
STAGE: Successful
PRODUCTION: Successful

Docker Image:
${ECR_URI}:${IMAGE_TAG}

Regards,
Jenkins CI/CD Pipeline
"""
            )
        }

        failure {
            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "FAILED: E-commerce CI/CD Build #${BUILD_NUMBER}",
                body: """
Hello,

Your E-commerce CI/CD pipeline has FAILED.

Build Number: ${BUILD_NUMBER}
Build URL: ${BUILD_URL}

Please check the Jenkins console output.

If deployment failed, automatic rollback was attempted on the affected server.

Regards,
Jenkins CI/CD Pipeline
"""
            )
        }

        aborted {
            emailext(
                to: "${NOTIFY_EMAIL}",
                subject: "ABORTED: E-commerce CI/CD Build #${BUILD_NUMBER}",
                body: """
Hello,

Your E-commerce CI/CD pipeline was ABORTED.

Build Number: ${BUILD_NUMBER}
Build URL: ${BUILD_URL}

Regards,
Jenkins CI/CD Pipeline
"""
            )
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
