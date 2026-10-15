
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
                    file(credentialsId: 'jobtrack-env',
                         variable: 'JOBTRACK_ENV')
                ]) {
                    sh '''
                        set -eu
                        docker compose --env-file "$JOBTRACK_ENV" config -q
                    '''
                }
            }
        }

        stage('Backend Tests') {
            steps {
                sh '''
                    set -eu
                    docker build -t jobtrack-backend-ci ./backend
                    docker run --rm --network none \
                        -e DATABASE_URL=sqlite:///:memory: \
                        jobtrack-backend-ci \
                        python -m pytest app/tests -q
                '''
            }
        }

        stage('Build Images') {
            steps {
                withCredentials([
                    file(credentialsId: 'jobtrack-env',
                         variable: 'JOBTRACK_ENV')
                ]) {
                    sh '''
                        set -eu
                        docker compose --env-file "$JOBTRACK_ENV" \
                            build backend frontend
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([
                    file(credentialsId: 'jobtrack-env',
                         variable: 'JOBTRACK_ENV')
                ]) {
                    sh '''
                        set -eu
                        docker compose --env-file "$JOBTRACK_ENV" \
                            up -d --no-build
                        docker compose --env-file "$JOBTRACK_ENV" ps
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu
                    docker exec jobtrack_backend \
                        curl -fsS http://localhost:8000/
                    echo
                    docker exec jobtrack_frontend \
                        wget -q -O /dev/null http://localhost/
                    echo "Frontend health check passed"
                '''
            }
        }
    }

    post {
        success {
            echo 'JobTrack CI/CD completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Inspect the first failed stage.'
        }
        always {
            echo 'JobTrack pipeline finished.'
        }
    }
}
