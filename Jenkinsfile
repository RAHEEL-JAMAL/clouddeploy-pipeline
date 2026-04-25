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
                    APP_NAME = "app-" + env.BUILD_NUMBER
                    IMAGE_NAME = "raheeljamal/${APP_NAME}:v1"
                    APP_DIR = "/tmp/${APP_NAME}"

                    echo "🚀 App: ${APP_NAME}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branch = sh(
                        script: "git ls-remote --heads https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git | awk '{print \$2}' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim()

                    def selected = branch.split("\n")[0]
                    echo "✔ Selected branch: ${selected}"

                    env.REPO_BRANCH = selected
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh """
                    rm -rf ${APP_DIR}
                    git clone --depth 1 -b ${env.REPO_BRANCH} https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git ${APP_DIR}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def isNode = fileExists("${APP_DIR}/package.json")

                    if (isNode) {
                        echo "📦 Stack: node"
                    } else {
                        echo "📦 Stack: unknown (default node)"
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
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {

            sh '''
                set +x
                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                docker push raheeljamal/app-${BUILD_NUMBER}:v1
            '''
        }
    }
}
        stage('Deploy Container') {
            steps {
                script {

                    // detect port from image type
                    def dockerfileText = readFile("${APP_DIR}/Dockerfile").toLowerCase()

                    def containerPort = "3000"

                    if (dockerfileText.contains("nginx")) {
                        containerPort = "80"
                    } else if (dockerfileText.contains("expose 8000")) {
                        containerPort = "8000"
                    } else if (dockerfileText.contains("expose 5000")) {
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

                    APP_PORT = sh(script: "cat /tmp/${APP_NAME}_port.txt", returnStdout: true).trim()
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
