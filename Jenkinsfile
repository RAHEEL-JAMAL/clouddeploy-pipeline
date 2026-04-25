pipeline {
    agent any

    environment {
        IMAGE_NAME = "raheeljamal/${JOB_NAME}"
        DOCKER_CREDS = "dockerhub-cred"
        PORT = "80"
    }

    stages {

        stage("Init") {
            steps {
                script {
                    echo "🚀 Project: ${JOB_NAME}"
                }
            }
        }

        stage("Clone Repo") {
            steps {
                sh '''
                    rm -rf app
                    git clone --depth 1 https://github.com/RAHEEL-JAMAL/portfolio-website.git app
                '''
            }
        }

        stage("Build Docker Image") {
            steps {
                sh '''
                    cd app

                    cat > Dockerfile << 'EOF'
FROM node:20-alpine as build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine

# FIX: SPA routing + MIME issues
RUN rm -rf /usr/share/nginx/html/*
COPY --from=build /app/dist /usr/share/nginx/html

# FIX: proper SPA routing (prevents 404 refresh error)
RUN printf 'server {\n\
  listen 80;\n\
  server_name localhost;\n\
  root /usr/share/nginx/html;\n\
  index index.html;\n\
\n\
  location / {\n\
    try_files $uri /index.html;\n\
  }\n\
}\n' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

                    DOCKER_BUILDKIT=1 docker build -t $IMAGE_NAME:v1 .
                '''
            }
        }

        stage("Push Image") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDS}",
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push $IMAGE_NAME:v1
                    '''
                }
            }
        }

        stage("Deploy Container") {
            steps {
                sh '''
                    docker stop app || true
                    docker rm app || true

                    docker run -d \
                        --name app \
                        -p 80:80 \
                        $IMAGE_NAME:v1
                '''
            }
        }

        stage("Verify") {
            steps {
                echo "🚀 App deployed successfully"
                echo "👉 http://<your-vm-ip>"
            }
        }
    }

    post {
        success {
            echo "✅ DEPLOYMENT SUCCESS"
        }
        failure {
            echo "❌ DEPLOYMENT FAILED"
        }
    }
}
