pipeline {
    agent any

    environment {
        IMAGE_NAME = "jenkins-test-app"

        PROD_CONTAINER = "jenkins-test-container"
        TEMP_CONTAINER = "jenkins-test-new"

        PROD_PORT = "80"
        TEMP_PORT = "8081"
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
                docker build -t $IMAGE_NAME:latest .
                '''
            }
        }

        stage('Deploy New Version (Temp)') {
            steps {
                sh '''
                echo "🚀 Starting new version on temp port..."

                docker rm -f $TEMP_CONTAINER || true

                docker run -d \
                  --name $TEMP_CONTAINER \
                  -p $TEMP_PORT:80 \
                  $IMAGE_NAME:latest
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                echo "🩺 Running health check..."

                sleep 10

                for i in {1..5}; do
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

        stage('Switch Traffic to New Version') {
            steps {
                sh '''
                echo "🔄 Switching traffic to new version..."

                # STEP 1: Stop anything currently using port 80
                OLD_CONTAINER=$(docker ps --filter "publish=80" -q)

                if [ ! -z "$OLD_CONTAINER" ]; then
                    echo "Stopping old container using port 80: $OLD_CONTAINER"
                    docker stop $OLD_CONTAINER
                    docker rm $OLD_CONTAINER
                fi

                # STEP 2: Ensure old named container is removed
                docker rm -f $PROD_CONTAINER || true

                # STEP 3: Start NEW container on port 80
                docker run -d \
                  --name $PROD_CONTAINER \
                  --restart unless-stopped \
                  -p $PROD_PORT:80 \
                  $IMAGE_NAME:latest

                # STEP 4: Remove temp container
                docker rm -f $TEMP_CONTAINER || true

                echo "🎉 Successfully switched to new version on port 80"
                '''
            }
        }
    }

    post {

        success {
            echo "✅ Deployment SUCCESS: New version live on port 80"
        }

        failure {
            echo "❌ Deployment FAILED: Rolling back..."

            sh '''
            docker rm -f $TEMP_CONTAINER || true
            echo "Rollback complete - old version still running (if not replaced)."
            '''
        }
    }
}
