pipeline {
    agent any

    environment {
        DOCKER_USER = 'raheeljamal'
        DOCKER_IMAGE_BASE = 'raheeljamal'
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.APP_NAME = params.APP_NAME ?: "app-${env.BUILD_NUMBER}"
                    env.REPO_URL = params.REPO_URL

                    if (!env.REPO_URL) {
                        error("❌ REPO_URL is required")
                    }

                    env.CONTAINER_NAME = env.APP_NAME
                    env.WORKSPACE_DIR = "/tmp/${APP_NAME}"

                    echo "🚀 App: ${APP_NAME}"
                    echo "📦 Repo: ${REPO_URL}"
                }
            }
        }

        stage('Auto Detect Branch') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${REPO_URL}",
                        returnStdout: true
                    ).trim()

                    def list = branches.readLines().collect {
                        it.split()[1].replace("refs/heads/", "")
                    }

                    def selected = list.contains("main") ? "main" :
                                   list.contains("master") ? "master" :
                                   list[0]

                    env.BRANCH = selected
                    echo "✔ Selected branch: ${BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    sh """
                        rm -rf ${WORKSPACE_DIR}
                        git clone --depth 1 -b ${BRANCH} ${REPO_URL} ${WORKSPACE_DIR}
                    """
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("${WORKSPACE_DIR}/package.json")) {
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
                    def dockerfile = "${WORKSPACE_DIR}/Dockerfile"

                    if (!fileExists(dockerfile)) {
                        echo "⚠ No Dockerfile found → generating..."

                        writeFile file: dockerfile, text: """
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm","start"]
"""
                    } else {
                        echo "✔ Dockerfile already exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    env.IMAGE_NAME = "${DOCKER_IMAGE_BASE}/${APP_NAME}:v1"

                    sh """
                        cd ${WORKSPACE_DIR}
                        docker build -t ${IMAGE_NAME} .
                    """

                    echo "🐳 Built: ${IMAGE_NAME}"
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    script {
                        sh """
                            echo $PASS | docker login -u $USER --password-stdin
                            docker push ${IMAGE_NAME}
                        """
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {

                    def PORT = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true

                        docker run -d \
                        --name ${APP_NAME} \
                        -p ${PORT}:3000 \
                        ${IMAGE_NAME}
                    """

                    env.APP_PORT = PORT
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    echo "🚀 LIVE URL:"
                    echo "👉 http://192.168.122.127:${APP_PORT}"
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
