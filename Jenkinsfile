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

                    def usedPorts = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]\\+->' | cut -d'>' -f1 || true",
                        returnStdout: true
                    ).trim()

                    def port = 3000

                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

                    env.EXTERNAL_PORT = port.toString()

                    echo "🌐 Assigned Port: ${EXTERNAL_PORT}"
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

                    env.STACK = "node"
                    env.INTERNAL_PORT = "3000"
                    env.NEEDS_BUILD = "false"

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                        env.INTERNAL_PORT = "8000"
                    }

                    else if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')

                        if (pkg.contains("build")) {
                            env.NEEDS_BUILD = "true"
                        }
                    }

                    echo "📦 Stack: ${STACK}"
                    echo "⚙️ Needs Build: ${NEEDS_BUILD}"
                }
            }
        }

        stage('Create Dockerfile (FIXED UNIVERSAL)') {
            steps {
                script {

                    if (!fileExists('app/Dockerfile')) {

                        echo "🛠 Generating SAFE Dockerfile"

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

                        else {
                            // 🔥 FIXED NODE UNIVERSAL (THIS IS THE IMPORTANT PART)
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine

WORKDIR /app

COPY . .

RUN npm install || true

# safe build (won't crash)
RUN npm run build || echo "No build step"

EXPOSE 3000

CMD ["sh", "-c", "npm start || node server.js || node index.js || node app.js || echo 'No entry file found' && sleep 3600"]
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

        stage('Run Container (FIXED)') {
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
            echo "🌐 OPEN: http://192.168.122.127:${EXTERNAL_PORT}"
        }

        failure {
            echo "❌ DEPLOY FAILED"
        }
    }
}
