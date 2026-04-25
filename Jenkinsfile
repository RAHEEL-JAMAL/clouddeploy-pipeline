pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raheeljamal"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    def APP_NAME = params.APP_NAME ?: "app-${env.BUILD_NUMBER}"
                    def REPO_URL = params.REPO_URL

                    if (!REPO_URL) {
                        error("REPO_URL is required")
                    }

                    env.APP_NAME = APP_NAME
                    env.REPO_URL = REPO_URL
                    env.CONTAINER_NAME = APP_NAME

                    echo "🚀 App: ${APP_NAME}"
                    echo "📦 Repo: ${REPO_URL}"
                }
            }
        }

        stage('Detect Branch (AUTO)') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${REPO_URL}",
                        returnStdout: true
                    ).trim()

                    def branchList = branches.readLines().collect {
                        it.split()[1].replace("refs/heads/", "")
                    }

                    // AUTO SELECT FIRST AVAILABLE BRANCH
                    def selectedBranch = branchList.find { it == "main" } ? "main" : branchList[0]

                    env.BRANCH = selectedBranch

                    echo "✔ Auto branch: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    env.APP_DIR = "/tmp/${APP_NAME}"

                    sh """
                        rm -rf ${APP_DIR}
                        git clone --depth 1 -b ${BRANCH} ${REPO_URL} ${APP_DIR}
                    """
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
                    def dockerfile = "${APP_DIR}/Dockerfile"

                    if (!fileExists(dockerfile)) {
                        echo "⚠ Creating Dockerfile..."

                        writeFile file: dockerfile, text: """
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm","start"]
"""
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${DOCKER_IMAGE}/${APP_NAME}:v1"

                    sh """
                        cd ${APP_DIR}
                        docker build -t ${IMAGE_TAG} .
                    """

                    echo "🐳 Built: ${IMAGE_TAG}"
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
                        sh """
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker push ${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {

                    def PORT = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true

                        docker run -d \
                        -p ${PORT}:3000 \
                        --name ${APP_NAME} \
                        ${IMAGE_TAG}
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
