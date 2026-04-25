pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
        IMAGE_TAG = "latest"
        CONTAINER_NAME = "auto_container"
        PORT = "8000"
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
                    // You can also replace this with Jenkins parameter
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
                    } else if (fileExists('app/package.json')) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "unknown"
                    }

                    echo "📦 Detected Stack: ${env.STACK}"
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
                            npm run build || echo "No build step"
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
                            writeFile file: 'app/Dockerfile', text: '''
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
'''
                        }

                        else if (env.STACK == "node") {
                            writeFile file: 'app/Dockerfile', text: '''
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
'''
                        }

                        else {
                            writeFile file: 'app/Dockerfile', text: '''
FROM alpine
CMD echo "Unknown project"
'''
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

        stage('Run Container') {
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

                    else if (env.STACK == "node") {
                        sh '''
                            docker run -d \
                            --name auto_container \
                            -p 3000:3000 \
                            auto-app:latest
                        '''
                    }

                    else {
                        sh '''
                            docker run -d \
                            --name auto_container \
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
            echo "🌐 App should be running on VM IP with exposed port"
        }

        failure {
            echo "❌ DEPLOY FAILED — check logs"
        }
    }
}
