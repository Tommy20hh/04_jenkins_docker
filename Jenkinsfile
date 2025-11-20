pipeline {
    agent any

    environment {
        API_HOST = "http://192.168.56.1:3001"
    }

    parameters {
        booleanParam(
            name: 'CLEAN_VOLUMES',
            defaultValue: true,
            description: 'Clear database volumes'
        )
        string(
            name: 'API_HOST',
            defaultValue: 'http://192.168.56.1:3001',
            description: 'URL the frontend should call'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                echo "📦 Checking out source code..."
                checkout scm
            }
        }

        stage('Generate .env') {
            steps {
                echo "📝 Creating .env inside Jenkins workspace..."

                sh """
                    cat > .env <<EOF
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_DATABASE=attractions_db
MYSQL_USER=attractions_user
MYSQL_PASSWORD=attractions_pass
MYSQL_PORT=3306
PHPMYADMIN_PORT=8888
API_PORT=3001
DB_PORT=3306
FRONTEND_PORT=3000
API_HOST=${params.API_HOST}
EOF
                """

                echo ".env created."
            }
        }

        stage('Validate Compose') {
            steps {
                echo "🔍 Validating docker-compose.yml..."
                sh "docker compose config"
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "🚀 Deploying..."

                    def downCmd = "docker compose down"
                    if (params.CLEAN_VOLUMES) {
                        downCmd = "docker compose down -v"
                        echo "⚠️  Clearing volumes!"
                    }

                    sh downCmd
                    sh "docker compose build --no-cache"
                    sh "docker compose up -d"

                    echo "Deployment done."
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "⏳ Waiting for MySQL and API (can take up to 2 mins)..."
                    sh 'sleep 60'

                    sh "docker compose ps"

                    timeout(time: 180, unit: 'SECONDS') {
                        sh """
                            echo 'Checking API health...'
                            until curl -f http://localhost:3001/health; do
                                echo 'API not ready, retrying in 5s...'
                                sleep 5
                            done
                        """
                    }
                }
            }
        }



        stage('Verify') {
            steps {
                echo "🔎 Final verification..."
                sh """
                    docker compose ps
                    docker compose logs --tail=30
                """
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS!"
        }
        failure {
            echo "❌ Deployment failed."
            sh "docker compose logs --tail=50 || true"
        }
        always {
            echo "🧹 Cleaning unused Docker resources..."
            sh "docker image prune -f || true"
            sh "docker container prune -f || true"
        }
    }
}
