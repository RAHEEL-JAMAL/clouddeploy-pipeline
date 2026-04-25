pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raheeljamal/${APP_NAME}"
        CONTAINER_NAME = "${APP_NAME}"
        PORT_FILE = "/tmp/${APP_NAME}_port.txt"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    APP_NAME = params.APP_NAME ?: "app-${env.BUILD_NUMBER}"
                    REPO_URL = params.REPO_URL ?: "https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git"

                    echo "🚀 App: ${APP_NAME}"
                    echo "📦 Repo: ${REPO_URL}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${REPO_URL}",
                        returnStdout: true
                    ).trim()

                    def selectedBranch = branches.contains("main") ? "main" : "master"
                    env.BRANCH = selectedBranch

                    echo "✔ Auto-selected branch: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    sh """
                        rm -rf /tmp/${APP_NAME}
                        git clone --depth 1 -b ${BRANCH} ${REPO_URL} /tmp/${APP_NAME}
                    """
                    env.APP_DIR = "/tmp/${APP_NAME}"
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("${APP_DIR}/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "unknown"
                    }
                    echo "📦 Stack: ${STACK}"
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    def dockerfilePath = "${APP_DIR}/Dockerfile"

                    if (!fileExists(dockerfilePath)) {
                        echo "⚠ No Dockerfile found → generating optimized one..."

                        writeFile file: dockerfilePath, text: """
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev || npm install --omit=dev

COPY . .

EXPOSE 3000
CMD ["npm","start"]
"""
                        echo "✔ Dockerfile created"
                    } else {
                        echo "✔ Dockerfile already exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    sh """
                        cd ${APP_DIR}
                        docker build -t ${DOCKER_IMAGE}:v1 .
                    """
                    echo "🐳 Built: ${DOCKER_IMAGE}:v1"
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:v1
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {

                    // stop old container safely
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    """

                    // assign random port
                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker run -d \
                        -p ${port}:3000 \
                        --name ${CONTAINER_NAME} \
                        ${DOCKER_IMAGE}:v1
                    """

                    writeFile file: PORT_FILE, text: port
                    echo "🌐 App running on port: ${port}"
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    def port = readFile(PORT_FILE).trim()
                    echo "🚀 LIVE URL:"
                    echo "👉 http://192.168.122.127:${port}"
                }
            }
        }
    }

    post {
        success {
            echo "🚀 DEPLOYMENT SUCCESS"
        }

        failure {
            echo "❌ DEPLOYMENT FAILED"
        }
    }
}
