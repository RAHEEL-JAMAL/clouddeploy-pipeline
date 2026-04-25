pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "raheeljamal"
        IMAGE_NAME = "${DOCKERHUB_USER}/${APP_NAME}"
        CONTAINER_NAME = "${APP_NAME}"
    }

    parameters {
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'BRANCH', defaultValue: 'auto', description: 'Branch name (auto optional)')
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo "🚀 App: ${params.APP_NAME}"
                    echo "📦 Repo: ${params.REPO_URL}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branch = "main"
                    if (params.BRANCH == "auto") {
                        branch = sh(
                            script: "git ls-remote --heads ${params.REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##' | head -n 1",
                            returnStdout: true
                        ).trim()
                    } else {
                        branch = params.BRANCH
                    }
                    env.BRANCH = branch
                    echo "✔ Using branch: ${branch}"
                }
            }
        }

        stage('Clone Repo') {
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
                    def path = "/tmp/${APP_NAME}/Dockerfile"

                    if (!fileExists(path)) {
                        echo "⚠ No Dockerfile → creating default"

                        writeFile file: path, text: """
                        FROM nginx:alpine
                        COPY . /usr/share/nginx/html
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
                sh """
                    cd /tmp/${APP_NAME}
                    DOCKER_BUILDKIT=0 docker build --no-cache -t ${IMAGE_NAME}:v1 .
                """
                echo "🐳 Built: ${IMAGE_NAME}:v1"
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
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:v1
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true
                        docker run -d -p ${port}:80 --name ${APP_NAME} ${IMAGE_NAME}:v1
                    """

                    env.PORT = port
                    echo "🌐 Running on: http://192.168.122.127:${port}"
                }
            }
        }

        stage('Verify') {
            steps {
                echo "🚀 LIVE URL:"
                echo "👉 http://192.168.122.127:${PORT}"
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
