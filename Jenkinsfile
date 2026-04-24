pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "ayush2027/jenkins-test-app"
        IMAGE_TAG      = "${BUILD_NUMBER}"

        PROD_CONTAINER = "jenkins-test-container"
        TEMP_CONTAINER = "jenkins-test-new"

        PROD_PORT      = "80"
        TEMP_PORT      = "8081"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "🔨 Building Docker image..."
                    docker build -t $DOCKERHUB_REPO:$IMAGE_TAG .
                '''
            }
        }

        /* =========================
           🛡️ TRIVY SCAN ADDED HERE
        ========================== */
        stage('Trivy Scan (Non-blocking)') {
            steps {
                sh '''
                    echo "🛡️ Running Trivy security scan..."

                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v $WORKSPACE:/workspace \
                        aquasec/trivy image \
                        --severity CRITICAL,HIGH \
                        --exit-code 0 \
                        --format json \
                        --output /workspace/trivy-report.json \
                        $DOCKERHUB_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Login to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                sh '''
                    echo "📤 Pushing image to DockerHub..."
                    docker push $DOCKERHUB_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Pull Image') {
            steps {
                sh '''
                    echo "📥 Pulling image: $IMAGE_TAG"
                    docker pull $DOCKERHUB_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Deploy Temp Container') {
            steps {
                sh '''
                    echo "🚀 Starting temp container..."

                    docker rm -f $TEMP_CONTAINER || true

                    docker run -d \
                        --name $TEMP_CONTAINER \
                        -p $TEMP_PORT:80 \
                        $DOCKERHUB_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "🩺 Running health check..."
                    sleep 10

                    for i in $(seq 1 5); do
                        if docker exec $TEMP_CONTAINER curl -f http://localhost; then
                            echo "✅ Health check passed"
                            exit 0
                        fi
                        echo "⏳ Retry $i..."
                        sleep 5
                    done

                    echo "❌ Health check failed"
                    docker logs $TEMP_CONTAINER
                    docker rm -f $TEMP_CONTAINER
                    exit 1
                '''
            }
        }

        stage('Switch to Production') {
            steps {
                sh '''
                    echo "🔄 Switching traffic to new version..."

                    docker rm -f $PROD_CONTAINER || true

                    docker run -d \
                        --name $PROD_CONTAINER \
                        --restart unless-stopped \
                        -p $PROD_PORT:80 \
                        $DOCKERHUB_REPO:$IMAGE_TAG

                    docker rm -f $TEMP_CONTAINER || true

                    echo "🎉 Deployment successful: version $IMAGE_TAG"
                '''
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: Version ${IMAGE_TAG} deployed successfully"
            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
        }

        failure {
            echo "❌ FAILED: Cleaning up temp container"
            sh 'docker rm -f jenkins-test-new || true'
        }
    }
}
