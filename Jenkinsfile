
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        COMPOSE_PROJECT_NAME = 'jobtrack'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Compose') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'jobtrack-env',
                        variable: 'JOBTRACK_ENV'
                    )
                ]) {
                    sh '''
                        set -eu
                        echo "Validating Docker Compose configuration..."
                        docker compose --env-file "$JOBTRACK_ENV" config -q
                        echo "Docker Compose configuration is valid."
                    '''
                }
            }
        }

        stage('Backend Tests') {
            steps {
                sh '''
                    set -eu

                    echo "Building backend CI test image..."
                    docker build -t jobtrack-backend-ci ./backend

                    echo "Running backend tests..."
                    docker run --rm \
                        --network none \
                        -e DATABASE_URL=sqlite:///:memory: \
                        jobtrack-backend-ci \
                        python -m pytest app/tests -q
                '''
            }
        }

        stage('Build Images') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'jobtrack-env',
                        variable: 'JOBTRACK_ENV'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Building JobTrack backend and frontend images..."
                        docker compose --env-file "$JOBTRACK_ENV" build backend frontend

                        echo "Docker images built successfully."
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'jobtrack-env',
                        variable: 'JOBTRACK_ENV'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Deploying JobTrack services..."
                        docker compose --env-file "$JOBTRACK_ENV" up -d --no-build

                        echo "Current JobTrack container status:"
                        docker compose --env-file "$JOBTRACK_ENV" ps
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    echo "Checking backend health..."
                    docker exec jobtrack_backend \
                        curl -fsS http://localhost:8000/
                    echo
                    echo "Backend health check passed."

                    echo "Checking frontend health..."

                    attempt=1
                    max_attempts=12

                    until docker exec jobtrack_frontend \
                        wget -q -O /dev/null http://127.0.0.1/; do

                        if [ "$attempt" -ge "$max_attempts" ]; then
                            echo "Frontend health check failed after retries."
                            echo "Frontend container logs:"
                            docker logs --tail 50 jobtrack_frontend
                            exit 1
                        fi

                        echo "Frontend not ready. Attempt $attempt/$max_attempts."
                        sleep 5
                        attempt=$((attempt + 1))
                    done

                    echo "Frontend health check passed."
                '''
            }
        }
    }

    post {
        success {
            echo 'JobTrack CI/CD completed successfully.'
            echo 'Frontend: http://localhost:3016'
            echo 'Backend API: http://localhost:8016'
            echo 'API documentation: http://localhost:8016/docs'
        }

        failure {
            echo 'JobTrack pipeline failed. Inspect the first failed stage in the console output.'
        }

        always {
            echo 'JobTrack pipeline finished.'
        }
    }
}
