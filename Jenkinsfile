pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 MULTI APP SAFE DEPLOY STARTED"
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    env.REPO_URL = params.REPO_URL
                    env.APP_ID = sh(script: "echo ${REPO_URL} | md5sum | cut -c1-6", returnStdout: true).trim()
                    env.CONTAINER_NAME = "app_${APP_ID}"

                    echo "📦 Repo: ${REPO_URL}"
                    echo "🆔 App ID: ${APP_ID}"
                }
            }
        }

        stage('Allocate Safe Port (FIXED)') {
            steps {
                script {

                    def usedPorts = sh(
                        script: """
                            docker ps --format '{{.Ports}}' \
                            | grep -o '[0-9]\\+->' \
                            | cut -d'>' -f1 || true
                        """,
                        returnStdout: true
                    ).trim()

                    def port = 3000

                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

                    env.EXTERNAL_PORT = port.toString()

                    echo "🌐 Assigned Safe Port: ${EXTERNAL_PORT}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh '''
                    rm -rf app
                    git clone --depth 1 ${REPO_URL} app
                '''
            }
        }

        stage('Detect Stack') {
            steps {
                script {

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                        env.INTERNAL_PORT = "8000"

                    } else if (fileExists('app/package.json')) {
                        env.STACK = "node"
                        env.INTERNAL_PORT = "3000"

                    } else if (fileExists('app/index.php')) {
                        env.STACK = "php"
                        env.INTERNAL_PORT = "80"

                    } else {
                        env.STACK = "unknown"
                        env.INTERNAL_PORT = "80"
                    }

                    echo "📦 Stack: ${STACK}"
                    echo "🔌 Internal Port: ${INTERNAL_PORT}"
                }
            }
        }

        stage('Create Dockerfile if missing (FIXED UNIVERSAL)') {
            steps {
                script {

                    if (!fileExists('app/Dockerfile')) {

                        echo "🛠 Creating AUTO Dockerfile"

                        if (env.STACK == "django") {
                            writeFile file: 'app/Dockerfile', text: """
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                        }

                        else if (env.STACK == "node") {
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install

EXPOSE 3000

CMD ["npm", "start"]
"""
                        }

                        else if (env.STACK == "php") {
                            writeFile file: 'app/Dockerfile', text: """
FROM php:8.2-apache
COPY . /var/www/html
EXPOSE 80
"""
                        }

                        else {
                            writeFile file: 'app/Dockerfile', text: """
FROM alpine
CMD echo "Unknown project"
"""
                        }
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    cd app
                    docker build -t ${IMAGE_NAME}:${APP_ID} .
                '''
            }
        }

        stage('Stop Only Same App') {
            steps {
                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''
            }
        }

        stage('Run Container (FULLY PARALLEL SAFE)') {
            steps {
                script {

                    if (env.STACK == "django") {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:8000 \
                            ${IMAGE_NAME}:${APP_ID}
                        """
                    }

                    else {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:3000 \
                            ${IMAGE_NAME}:${APP_ID}
                        """
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh 'docker ps'
            }
        }
    }

    post {
        success {
            echo "✅ DEPLOY SUCCESS"
            echo "🌐 OPEN → http://192.168.122.127:${EXTERNAL_PORT}"
        }

        failure {
            echo "❌ DEPLOY FAILED"
        }
    }
}
