pipeline {
    agent any

    parameters {
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch name')
    }

    environment {
        DOCKERHUB_USER = "raheeljamal"
        IMAGE_NAME = "${DOCKERHUB_USER}/${params.APP_NAME}"
        CONTAINER_NAME = "${params.APP_NAME}"
        DEPLOY_PORT = "8080"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 Deploying: ${params.APP_NAME}"
                echo "📦 Repo: ${params.REPO_URL}"
            }
        }

        stage('Clone Repo') {
            steps {
                sh """
                    rm -rf app
                    git clone --depth 1 -b ${params.BRANCH} ${params.REPO_URL} app
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("app/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "static"
                    }
                    echo "📦 Stack detected: ${STACK}"
                }
            }
        }

        stage('Build Frontend (SAFE VITE FIX)') {
            steps {
                script {
                    if (env.STACK == "node") {
                        sh """
                            cd app
                            npm install
                            npm run build
                        """
                    }
                }
            }
        }

        stage('Prepare Dockerfile (FIXED)') {
            steps {
                script {
                    def dockerfilePath = "app/Dockerfile"

                    if (!fileExists(dockerfilePath)) {

                        if (env.STACK == "node") {
                            writeFile file: dockerfilePath, text: """
FROM nginx:alpine
COPY dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                        } else {
                            writeFile file: dockerfilePath, text: """
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                        }
                    }
                }
            }
        }

        stage('Build Docker Image (SAFE MODE)') {
            steps {
                sh """
                    cd app
                    docker build --no-cache -t ${IMAGE_NAME}:v1 .
                """
            }
        }

        stage('Push Image (FIXED CREDENTIALS)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:v1
                    """
                }
            }
        }

        stage('Deploy Container (FIXED PORT)') {
            steps {
                script {
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                        docker run -d -p ${DEPLOY_PORT}:80 --name ${CONTAINER_NAME} ${IMAGE_NAME}:v1
                    """

                    echo "🌐 App Running:"
                    echo "👉 http://192.168.122.127:${DEPLOY_PORT}"
                }
            }
        }

        stage('Verify') {
            steps {
                sh "docker ps"
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
