pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
        IMAGE_TAG = "latest"
        CONTAINER_NAME = "auto_container"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 AUTO DEPLOY PIPELINE STARTED"
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    env.REPO_URL = params.REPO_URL ?: "https://github.com/RAHEEL-JAMAL/LondheShubham153-django-notes-app.git"
                    echo "📦 Repo: ${env.REPO_URL}"
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
                    } 
                    else if (fileExists('app/package.json')) {
                        env.STACK = "node"
                        env.INTERNAL_PORT = "80"
                    } 
                    else if (fileExists('app/index.php')) {
                        env.STACK = "php"
                        env.INTERNAL_PORT = "80"
                    }
                    else {
                        env.STACK = "unknown"
                        env.INTERNAL_PORT = "80"
                    }

                    echo "📦 Stack: ${env.STACK}"
                    echo "🔌 Internal Port: ${env.INTERNAL_PORT}"
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
                            npm run build || echo "No build step required"
                        '''
                    } else {
                        echo "ℹ️ No build step required"
                    }
                }
            }
        }

        stage('Create Dockerfile if missing') {
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

                        else {
                            writeFile file: 'app/Dockerfile', text: """
FROM alpine
CMD echo "Unknown project"
"""
                        }
                    } else {
                        echo "📝 Dockerfile already exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    cd app
                    export DOCKER_BUILDKIT=0
                    docker build -t auto-app:latest .
                '''
            }
        }

        stage('Stop Old Container') {
            steps {
                sh '''
                    docker stop auto_container || true
                    docker rm auto_container || true
                '''
            }
        }

        stage('Run Container (FIXED PORT LOGIC)') {
            steps {
                script {

                    if (env.STACK == "django") {
                        sh '''
                            docker run -d \
                            --name auto_container \
                            -p 8000:8000 \
                            auto-app:latest
                        '''
                    }

                    else {
                        // IMPORTANT FIX: React + PHP + Nginx all use port 80 inside container
                        sh '''
                            docker run -d \
                            --name auto_container \
                            -p 3000:80 \
                            auto-app:latest
                        '''
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "🔍 Running Containers:"
                    docker ps
                '''
            }
        }

    }

    post {
        success {
            echo "✅ AUTO DEPLOY SUCCESS"
            echo "🌐 Django → http://VM_IP:8000"
            echo "🌐 Other Apps → http://VM_IP:3000"
        }

        failure {
            echo "❌ DEPLOY FAILED — check logs"
        }
    }
}
