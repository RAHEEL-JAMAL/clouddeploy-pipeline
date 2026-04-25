pipeline {
    agent any

    environment {
        DOCKER_BUILDKIT = "0"
        REGISTRY = "raheeljamal"
        PORT = ""
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.APP_NAME = params.APP_NAME ?: "myapp"
                    env.REPO_URL = params.REPO_URL
                    echo "🚀 App: ${APP_NAME}"
                    echo "📦 Repo: ${REPO_URL}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branch = sh(
                        script: "git ls-remote --heads ${REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##' | head -n 1",
                        returnStdout: true
                    ).trim()

                    env.BRANCH = branch ?: "main"
                    echo "✔ Using branch: ${BRANCH}"
                }
            }
        }

        stage('Clone Repo (FAST)') {
            steps {
                sh """
                    rm -rf /tmp/${APP_NAME}
                    git clone --depth 1 -b ${BRANCH} ${REPO_URL} /tmp/${APP_NAME}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("/tmp/${APP_NAME}/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "static"
                    }
                    echo "📦 Stack: ${STACK}"
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    def dockerfile = "/tmp/${APP_NAME}/Dockerfile"

                    if (!fileExists(dockerfile)) {
                        echo "⚠ No Dockerfile → creating default"
                        writeFile file: dockerfile, text: """
FROM node:20-alpine as build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                    } else {
                        echo "✔ Dockerfile already exists"
                    }
                }
            }
        }

        stage('Build Image (FAST FIXED)') {
            steps {
                sh """
                    cd /tmp/${APP_NAME}
                    DOCKER_BUILDKIT=0 docker build -t ${REGISTRY}/${APP_NAME}:v${BUILD_NUMBER} .
                """
                echo "🐳 Image built"
            }
        }

        stage('Push Image (SAFE LOGIN FIXED)') {
            steps {
                withCredentials([string(credentialsId: 'docker-pass', variable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u raheeljamal --password-stdin
                        docker push raheeljamal/${APP_NAME}:v${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Deploy Container (FAST PORT AUTO)') {
            steps {
                script {
                    env.PORT = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true

                        docker run -d \
                        -p ${PORT}:80 \
                        --name ${APP_NAME} \
                        ${REGISTRY}/${APP_NAME}:v${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Verify') {
            steps {
                echo "🚀 DEPLOY SUCCESS"
                echo "👉 LIVE URL: http://192.168.122.127:${PORT}"
            }
        }
    }

    post {
        success {
            echo "🚀 Pipeline SUCCESS"
        }
        failure {
            echo "❌ Pipeline FAILED"
        }
    }
}
