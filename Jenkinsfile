pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo "[STAGE_START] Init"
                    echo "🚀 MULTI APP SAFE DEPLOY STARTED"
                    echo "[STAGE_SUCCESS] Init"
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    echo "[STAGE_START] Input Repo"

                    env.REPO_URL = params.REPO_URL
                    env.APP_ID = sh(script: "echo ${REPO_URL} | md5sum | cut -c1-6", returnStdout: true).trim()
                    env.CONTAINER_NAME = "app_${APP_ID}"

                    echo "[META] REPO_URL=${REPO_URL}"
                    echo "[META] APP_ID=${APP_ID}"
                    echo "[META] CONTAINER_NAME=${CONTAINER_NAME}"

                    echo "[STAGE_SUCCESS] Input Repo"
                }
            }
        }

        stage('Allocate Safe Port') {
            steps {
                script {
                    echo "[STAGE_START] Allocate Safe Port"

                    def usedPorts = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | cut -d'-' -f1 || true",
                        returnStdout: true
                    ).trim()

                    def port = 3000
                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

                    env.EXTERNAL_PORT = port.toString()

                    echo "[META] PORT=${EXTERNAL_PORT}"
                    echo "[STAGE_SUCCESS] Allocate Safe Port"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    echo "[STAGE_START] Clone Repo"
                }

                sh '''
                    rm -rf app
                    git clone --depth 1 ${REPO_URL} app
                '''

                script {
                    echo "[STAGE_SUCCESS] Clone Repo"
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo "[STAGE_START] Detect Stack"

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                    } else if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')
                        if (pkg.contains("vite")) {
                            env.STACK = "vite"
                        } else {
                            env.STACK = "node"
                        }
                    } else {
                        env.STACK = "node"
                    }

                    echo "[META] STACK=${STACK}"
                    echo "[STAGE_SUCCESS] Detect Stack"
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    echo "[STAGE_START] Create Dockerfile"

                    if (!fileExists('app/Dockerfile')) {

                        if (env.STACK == "django") {
                            writeFile file: 'app/Dockerfile', text: """
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt || true
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                        } else if (env.STACK == "vite") {
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
                        } else {
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install || true
ENV PORT=3000
EXPOSE 3000
CMD sh -c "PORT=3000 node server.js || PORT=3000 node index.js || PORT=3000 npm start || PORT=3000 node app.js"
"""
                        }
                    }

                    echo "[STAGE_SUCCESS] Create Dockerfile"
                }
            }
        }

        stage('Build Image') {
            steps {
                script { echo "[STAGE_START] Build Image" }

                sh '''
                    cd app
                    docker build -t auto-app:${APP_ID} .
                '''

                script { echo "[STAGE_SUCCESS] Build Image" }
            }
        }

        // ─── NEW STAGE ────────────────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script {
                    echo "[STAGE_START] Push to DockerHub"
                }

                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                            # Tag image as dockerhub_user/auto-app:APP_ID
                            docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:${APP_ID}

                            # Also tag as latest for convenience
                            docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:latest

                            # Push both tags
                            docker push ${DOCKER_USER}/auto-app:${APP_ID}
                            docker push ${DOCKER_USER}/auto-app:latest

                            echo "[META] DOCKER_IMAGE=${DOCKER_USER}/auto-app:${APP_ID}"
                        '''
                    }
                }

                script {
                    echo "[STAGE_SUCCESS] Push to DockerHub"
                }
            }
        }
        // ─────────────────────────────────────────────────────────────────────

        stage('Stop Old Container') {
            steps {
                script { echo "[STAGE_START] Stop Old Container" }

                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''

                script { echo "[STAGE_SUCCESS] Stop Old Container" }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    echo "[STAGE_START] Run Container"

                    if (env.STACK == "django") {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:8000 \
                        auto-app:${APP_ID}
                        """
                    } else if (env.STACK == "vite") {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:80 \
                        auto-app:${APP_ID}
                        """
                    } else {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:3000 \
                        auto-app:${APP_ID}
                        """
                    }

                    echo "[STAGE_SUCCESS] Run Container"
                }
            }
        }

        stage('Verify') {
            steps {
                script { echo "[STAGE_START] Verify" }

                sh 'docker ps'

                script {
                    echo "[META] URL=http://192.168.122.127:${EXTERNAL_PORT}"
                    echo "[STAGE_SUCCESS] Verify"
                }
            }
        }
    }

    post {
        success {
            echo "[DEPLOY_SUCCESS]"
            echo "[META] FINAL_STATUS=success"
            echo "[META] URL=http://192.168.122.127:${EXTERNAL_PORT}"
        }

        failure {
            echo "[DEPLOY_FAILED]"
            echo "[META] FINAL_STATUS=failed"
        }
    }
}
