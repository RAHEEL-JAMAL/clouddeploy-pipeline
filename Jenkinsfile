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
                    env.REPO_URL = params.REPO_URL

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

                    def branchList = branches.readLines().collect {
                        it.split()[1].replace("refs/heads/", "")
                    }

                    env.BRANCH = branchList.contains("main") ? "main" :
                                 branchList.contains("master") ? "master" :
                                 branchList[0]

                    echo "✔ Auto branch: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    sh "rm -rf /tmp/${env.APP_NAME}"
                    sh "git clone --depth 1 -b ${env.BRANCH} ${env.REPO_URL} /tmp/${env.APP_NAME}"
                    env.APP_DIR = "/tmp/${env.APP_NAME}"
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    env.STACK = fileExists("${env.APP_DIR}/package.json") ? "node" : "unknown"
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
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev || npm install --omit=dev

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
                    env.IMAGE_TAG = "v${env.BUILD_NUMBER}"

                    sh """
                        cd ${env.APP_DIR}
                        DOCKER_BUILDKIT=1 docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    """

                    echo "🐳 Built: ${DOCKER_IMAGE}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push Image (FIXED SAFE LOGIN)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''

                        sh """
                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                script {

                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    """

                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker run -d \
                        -p ${port}:3000 \
                        --name ${CONTAINER_NAME} \
                        ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """

                    env.PORT = port
                }
            }
        }

        stage('Verify') {
            steps {
                echo "🚀 LIVE URL:"
                echo "👉 http://192.168.122.127:${env.PORT}"
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
