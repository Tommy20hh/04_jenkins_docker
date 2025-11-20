pipeline {
    agent any

    environment {
        MYSQL_ROOT_PASSWORD = "rootpassword"
        MYSQL_DATABASE = "attractions_db"
        MYSQL_USER = "attractions_user"
        MYSQL_PASSWORD = "attractions_pass"
        MYSQL_PORT = "3306"

        PHPMYADMIN_PORT = "8888"

        API_PORT = "3001"
        DB_PORT = "3306"
        FRONTEND_PORT = "3000"
    }

    parameters {
        booleanParam(
            name: 'CLEAN_VOLUMES',
            defaultValue: true,
            description: 'Remove volumes (clears database)'
        )

        string(
            name: 'API_HOST',
            defaultValue: 'http://192.168.56.1:3001',
            description: 'API base URL for frontend'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out SCM..."
                checkout scm
            }
        }

        stage('Validate Compose') {
            steps {
                echo "Validating Docker Compose file..."
                sh 'docker compose config'
            }
        }

        stage('Create .env file') {
            steps {
                script {
                    echo "Generating .env file..."

                    sh """
                        cat > .env <<EOF
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
MYSQL_DATABASE=${MYSQL_DATABASE}
MYSQL_USER=${MYSQL_USER}
MYSQL_PASSWORD=${MYSQL_PASSWORD}
MYSQL_PORT=${MYSQL_PORT}

PHPMYADMIN_PORT=${PHPMYADMIN_PORT}

API_PORT=${API_PORT}
DB_PORT=${DB_PORT}
FRONTEND_PORT=${FRONTEND_PORT}

NODE_ENV=production
API_HOST=${params.API_HOST}
EOF
                    """

                    echo ".env file created."
                }
            }
        }

        stage('Deploy') {
            steps {
                script {

                    echo "Stopping old containers..."

                    if (params.CLEAN_VOLUMES) {
                        sh 'docker compose down -v'
                    } else {
                        sh 'docker compose down'
                    }

                    echo "Building fresh images..."
                    sh 'docker compose build --no-cache'

                    echo "Starting containers..."
                    sh 'docker compose up -d'
                }
            }
        }

        stage('Health Check') {
            steps {
                script {

                    echo "Waiting for API to start..."
                    sh 'sleep 15'

                    echo "Checking API health endpoint..."

                    sh """
                        timeout 60 bash -c 'until curl -fs http://localhost:${API_PORT}/health; do sleep 3; done'
                        curl -f http://localhost:${API_PORT}/attractions
                    """

                    echo "Health check PASSED"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Showing running containers..."
                sh "docker compose ps"

                echo "Showing service logs..."
                sh "docker compose logs --tail=20"
            }
        }
    }

    post {
        success {
            echo "🚀 SUCCESS! Deployment completed!"
        }

        failure {
            echo "❌ Deployment failed! Showing last logs..."
            sh "docker compose logs --tail=50 || true"
        }

        always {
            sh "docker image prune -f"
            sh "docker container prune -f"
        }
    }
}
