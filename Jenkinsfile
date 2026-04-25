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

        stage('Allocate Safe Port') {
            steps {
                script {

                    def used = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | cut -d'-' -f1 || true",
                        returnStdout: true
                    ).trim()

                    def port = 3000

                    while (used.contains(port.toString())) {
                        port++
                    }

                    env.EXTERNAL_PORT = port.toString()
                    echo "🌐 SAFE PORT: ${EXTERNAL_PORT}"
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

        stage('Detect Stack (FIXED LOGIC)') {
            steps {
                script {

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"

                    } else if (fileExists('app/package.json')) {
                        
                        def pkg = readFile('app/package.json')

                        if (pkg.contains("vite") || fileExists('app/vite.config.js')) {
                            env.STACK = "vite"
                        } else {
                            env.STACK = "node"
                        }

                    } else if (fileExists('app/index.php')) {
                        env.STACK = "php"

                    } else {
                        env.STACK = "node"
                    }

                    echo "📦 STACK: ${STACK}"
                }
            }
        }

        stage('Create Dockerfile (FIXED UNIVERSAL)') {
            steps {
                script {

                    if (!fileExists('app/Dockerfile')) {

                        // 🔵 DJANGO
                        if (env.STACK == "django") {
                            writeFile file: 'app/Dockerfile', text: """
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt || true
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                        }

                        // 🟢 NODE SERVER (chatcord, express, socket.io)
                        else if (env.STACK == "node") {
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install || true
EXPOSE 3000
CMD ["npm", "start"]
"""
                        }

                        // 🟡 VITE / REACT STATIC
                        else if (env.STACK == "vite") {
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm install || true
RUN npm run build || true

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                        }

                        // 🟠 PHP
                        else {
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
                    docker build -t auto-app:${APP_ID} .
                '''
            }
        }

        stage('Stop Old Container') {
            steps {
                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''
            }
        }

        stage('Run Container (FIXED PORT LOGIC)') {
            steps {
                script {

                    if (env.STACK == "django") {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:8000 \
                            auto-app:${APP_ID}
                        """
                    }

                    else if (env.STACK == "vite") {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:80 \
                            auto-app:${APP_ID}
                        """
                    }

                    else {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:3000 \
                            auto-app:${APP_ID}
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
            echo "🌐 OPEN IN BROWSER → http://YOUR_VM_IP:${EXTERNAL_PORT}"
        }

        failure {
            echo "❌ DEPLOY FAILED"
        }
    }
}
