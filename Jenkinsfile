pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raheeljamal"
        CONTAINER_NAME = ""
        PORT_FILE = ""
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.APP_NAME = params.APP_NAME ?: "app-${env.BUILD_NUMBER}"
                    env.REPO_URL = params.REPO_URL ?: "https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git"

                    env.CONTAINER_NAME = env.APP_NAME
                    env.PORT_FILE = "/tmp/${env.APP_NAME}_port.txt"

                    echo "🚀 App: ${env.APP_NAME}"
                    echo "📦 Repo: ${env.REPO_URL}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim()

                    def branchList = branches.split("\n")
                    env.BRANCH = branchList[0]

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
                    env.IMAGE = "${DOCKER_IMAGE}/${APP_NAME}:v${BUILD_NUMBER}"

                    sh """
                        cd ${APP_DIR}
                        docker build -t ${IMAGE} .
                    """

                    echo "🐳 Built: ${IMAGE}"
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${IMAGE}
                        '''
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true
                    """

                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()
                    env.PORT = port

                    sh """
                        docker run -d \
                        -p ${PORT}:3000 \
                        --name ${APP_NAME} \
                        ${IMAGE}
                    """

                    writeFile file: PORT_FILE, text: "${PORT}"
                    echo "🌐 Running on port: ${PORT}"
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
