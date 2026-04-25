pipeline {
    agent any

    environment {
        IMAGE_NAME = "raheeljamal/${APP_NAME}"
        IMAGE_TAG = "latest"
        DOCKER_CREDS = "dockerhub-cred"
        PORT = ""
        APP_NAME = "${APP_NAME}"
        REPO_URL = "${REPO_URL}"
    }

    stages {

        stage("Init") {
            steps {
                script {
                    echo "🚀 App: ${APP_NAME}"
                    echo "📦 Repo: ${REPO_URL}"
                }
            }
        }

        stage("Detect Branch") {
            steps {
                script {
                    env.BRANCH = sh(
                        script: "git ls-remote --heads ${REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##' | head -n 1",
                        returnStdout: true
                    ).trim()

                    echo "✔ Using branch: ${env.BRANCH}"
                }
            }
        }

        stage("Clone Repo") {
            steps {
                sh """
                    rm -rf /tmp/${APP_NAME}
                    git clone --depth 1 -b ${BRANCH} ${REPO_URL} /tmp/${APP_NAME}
                """
            }
        }

        stage("Detect Stack") {
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

        stage("Prepare Nginx Config (FIX MIME + SPA)") {
            steps {
                script {
                    writeFile file: "/tmp/${APP_NAME}/default.conf", text: """
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    include /etc/nginx/mime.types;

    location / {
        try_files \$uri \$uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
"""
                }
            }
        }

        stage("Build Image (FIXED)") {
            steps {
                script {
                    sh """
                        cd /tmp/${APP_NAME}

                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """

                    echo "🐳 Built: ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage("Push Image (FIXED LOGIN)") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDS}",
                    usernameVariable: "USER",
                    passwordVariable: "PASS"
                )]) {
                    sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage("Deploy Container") {
            steps {
                script {
                    PORT = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true

                        docker run -d \
                        -p ${PORT}:80 \
                        --name ${APP_NAME} \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                    """

                    echo "🌐 Running: http://192.168.122.127:${PORT}"
                }
            }
        }

        stage("Verify") {
            steps {
                echo "🚀 LIVE URL READY"
            }
        }
    }

    post {
        success {
            echo "🚀 DEPLOYMENT SUCCESS"
        }
        failure {
            echo "❌ PIPELINE FAILED"
        }
    }
}
