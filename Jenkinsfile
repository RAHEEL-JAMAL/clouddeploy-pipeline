pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
        IMAGE_TAG = "latest"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 MULTI APP AUTO DEPLOY STARTED"
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    env.REPO_URL = params.REPO_URL
                    env.APP_ID = sh(script: "echo ${REPO_URL} | md5sum | cut -c1-6", returnStdout: true).trim()

                    echo "📦 Repo: ${env.REPO_URL}"
                    echo "🆔 App ID: ${env.APP_ID}"
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

        stage('Detect Stack + Port') {
            steps {
                script {

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                        env.INTERNAL_PORT = "8000"
                        env.EXTERNAL_PORT = "8000"

                    } else if (fileExists('app/package.json')) {
                        env.STACK = "node"
                        env.INTERNAL_PORT = "80"

                    } else if (fileExists('app/index.php')) {
                        env.STACK = "php"
                        env.INTERNAL_PORT = "80"

                    } else {
                        env.STACK = "unknown"
                        env.INTERNAL_PORT = "80"
                    }

                    // AUTO PORT GENERATION (NO CONFLICT)
                    env.EXTERNAL_PORT = sh(
                        script: "echo \$((3000 + RANDOM % 500))",
                        returnStdout: true
                    ).trim()

                    env.CONTAINER_NAME = "app_${APP_ID}"

                    echo "📦 Stack: ${STACK}"
                    echo "🔌 Internal: ${INTERNAL_PORT}"
                    echo "🌐 External: ${EXTERNAL_PORT}"
                    echo "📦 Container: ${CONTAINER_NAME}"
                }
            }
        }

        stage('Build App') {
            steps {
                script {
                    if (env.STACK == "node") {
                        sh '''
                            cd app
                            npm install
                            npm run build || true
                        '''
                    } else {
                        echo "ℹ️ No build required"
                    }
                }
            }
        }

        stage('Create Dockerfile if missing') {
            steps {
                script {

                    if (!fileExists('app/Dockerfile')) {

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
FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                        }

                        else if (env.STACK == "php") {
                            writeFile file: 'app/Dockerfile', text: """
FROM php:8.2-apache
COPY . /var/www/html
EXPOSE 80
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

        stage('Stop Old Container (Same App Only)') {
            steps {
                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''
            }
        }

        stage('Run Container (MULTI APP SAFE)') {
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
                            -p ${EXTERNAL_PORT}:80 \
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
            echo "🌐 App URL: http://VM_IP:${EXTERNAL_PORT}"
        }

        failure {
            echo "❌ FAILED"
        }
    }
}
