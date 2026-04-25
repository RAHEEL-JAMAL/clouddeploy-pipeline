pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "raheeljamal/${APP_NAME}"
        CONTAINER_NAME = "${APP_NAME}"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.APP_NAME = params.APP_NAME ?: "app-${env.BUILD_NUMBER}"
                    env.REPO_URL = params.REPO_URL ?: "https://github.com/RAHEEL-JAMAL/dockerizing-reactjs-app.git"

                    echo "🚀 App: ${env.APP_NAME}"
                    echo "📦 Repo: ${env.REPO_URL}"
                }
            }
        }

        stage('Detect Branch (AUTO)') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${env.REPO_URL}",
                        returnStdout: true
                    ).trim()

                    def selected = branches.contains("main") ? "main" : "master"
                    env.BRANCH = selected

                    echo "✔ Auto branch: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    env.APP_DIR = "/tmp/${env.APP_NAME}"

                    sh """
                        rm -rf ${env.APP_DIR}
                        git clone --depth 1 -b ${env.BRANCH} ${env.REPO_URL} ${env.APP_DIR}
                    """
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("${env.APP_DIR}/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "unknown"
                    }

                    echo "📦 Stack: ${env.STACK}"
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    def dockerfile = "${env.APP_DIR}/Dockerfile"

                    if (!fileExists(dockerfile)) {
                        echo "⚠ No Dockerfile → creating"

                        writeFile file: dockerfile, text: """
FROM node:20-alpine as build
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                    } else {
                        echo "✔ Dockerfile exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    sh """
                        cd ${env.APP_DIR}
                        docker build -t ${env.DOCKER_IMAGE}:v1 .
                    """

                    echo "🐳 Built: ${env.DOCKER_IMAGE}:v1"
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
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${env.DOCKER_IMAGE}:v1
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${env.CONTAINER_NAME} || true
                        docker rm ${env.CONTAINER_NAME} || true

                        docker run -d \
                        -p ${port}:80 \
                        --name ${env.CONTAINER_NAME} \
                        ${env.DOCKER_IMAGE}:v1
                    """

                    echo "🌐 Running on port: ${port}"
                    env.PORT = port
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    echo "🚀 LIVE URL:"
                    echo "👉 http://192.168.122.127:${env.PORT}"
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
