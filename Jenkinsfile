pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: 'https://github.com/RAHEEL-JAMAL/portfolio-website.git', description: 'Git Repo URL')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App Name')
        string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Docker Image Tag')
    }

    environment {
        DOCKERHUB_USER = 'raheeljamal'
        IMAGE_NAME = "${DOCKERHUB_USER}/${params.APP_NAME}"
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
                    git clone --depth 1 ${params.REPO_URL} app
                """
            }
        }

        stage('Fix Frontend (SAFE NGINX ROUTING)') {
            steps {
                sh '''
                    cd app

                    # FIX: prevent vite/svg/mime issues
                    if [ ! -f Dockerfile ]; then
                        cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
                    fi
                '''
            }
        }

        stage('Build Image (STABLE MODE)') {
            steps {
                sh """
                    cd app

                    # IMPORTANT: disable BuildKit to avoid buildx issues
                    DOCKER_BUILDKIT=0 docker build -t ${IMAGE_NAME}:${params.IMAGE_TAG} .
                """
            }
        }

        stage('Push Image (FIXED CREDENTIALS)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push ${IMAGE_NAME}:${params.IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Container (CLEAN + FIXED PORT)') {
            steps {
                sh """
                    docker stop ${params.APP_NAME} || true
                    docker rm ${params.APP_NAME} || true

                    docker run -d \
                        --name ${params.APP_NAME} \
                        -p 80:80 \
                        ${IMAGE_NAME}:${params.IMAGE_TAG}
                """
            }
        }

        stage('Verify') {
            steps {
                echo "🌐 APP LIVE:"
                echo "👉 http://192.168.122.127"
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
