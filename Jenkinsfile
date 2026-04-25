pipeline {
    agent any

    environment {
        APP_NAME = ""
        IMAGE_NAME = ""
        APP_DIR = ""
        APP_PORT = ""
    }

    stages {

        stage('Init') {
            steps {
                script {
                    def app = "app-${env.BUILD_NUMBER}"
                    def image = "raheeljamal/${app}:v1"
                    def dir = "/tmp/${app}"

                    env.APP_NAME = app
                    env.IMAGE_NAME = image
                    env.APP_DIR = dir

                    echo "🚀 App: ${app}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branch = sh(
                        script: """
                            git ls-remote --heads https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git |
                            awk '{print \$2}' | sed 's#refs/heads/##'
                        """,
                        returnStdout: true
                    ).trim()

                    def selected = branch.split("\n")[0]
                    env.REPO_BRANCH = selected

                    echo "✔ Selected branch: ${selected}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh """
                    rm -rf ${APP_DIR}
                    git clone --depth 1 -b ${REPO_BRANCH} https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git ${APP_DIR}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("${APP_DIR}/package.json")) {
                        echo "📦 Stack: node"
                    }
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    if (!fileExists("${APP_DIR}/Dockerfile")) {
                        echo "⚠ No Dockerfile found → generating..."

                        def dockerfile = """
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev || npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["npm","start"]
"""

                        writeFile file: "${APP_DIR}/Dockerfile", text: dockerfile
                        echo "✔ Dockerfile created"
                    } else {
                        echo "✔ Dockerfile already exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh """
                    cd ${APP_DIR}
                    docker build -t ${IMAGE_NAME} .
                """
                echo "🐳 Built: ${IMAGE_NAME}"
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                        set +x
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {

                    def containerPort = "3000"

                    def df = readFile("${APP_DIR}/Dockerfile").toLowerCase()

                    if (df.contains("nginx")) {
                        containerPort = "80"
                    } else if (df.contains("expose 8000")) {
                        containerPort = "8000"
                    } else if (df.contains("expose 5000")) {
                        containerPort = "5000"
                    }

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true

                        PORT=\$(shuf -i 3000-3999 -n 1)

                        docker run -d \
                            -p \$PORT:${containerPort} \
                            --name ${APP_NAME} \
                            ${IMAGE_NAME}

                        echo \$PORT > /tmp/${APP_NAME}_port.txt
                    """

                    env.APP_PORT = sh(
                        script: "cat /tmp/${APP_NAME}_port.txt",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Verify') {
            steps {
                sh "docker ps | grep ${APP_NAME} || true"

                echo "🌍 LIVE URL:"
                echo "👉 http://192.168.122.127:${APP_PORT}"
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
