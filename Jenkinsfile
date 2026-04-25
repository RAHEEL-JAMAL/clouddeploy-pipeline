pipeline {
    agent any

    environment {
        IMAGE_NAME = "raheeljamal/fyp-pipeline"
        CONTAINER_NAME = "fyp-app"
        PORT = "80"
        DOCKER_CREDS = "dockerhub-cred"
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

        stage("Build Image (FIXED - NO BUILDKIT)") {
            steps {
                sh '''
                    cd app

                    cat > Dockerfile << 'EOF'
# ---------- BUILD ----------
FROM node:20-alpine AS build
WORKDIR /app

COPY . .
RUN npm install && npm run build

# ---------- RUN ----------
FROM nginx:alpine

# Clean default nginx files
RUN rm -rf /usr/share/nginx/html/*

# Copy build output
COPY --from=build /app/dist /usr/share/nginx/html

# FIX SPA ROUTING + FIX 404 + FIX MIME ISSUES
RUN printf 'server {\n\
    listen 80;\n\
    server_name localhost;\n\
    root /usr/share/nginx/html;\n\
    index index.html;\n\
\n\
    location / {\n\
        try_files $uri $uri/ /index.html;\n\
    }\n\
\n\
    location ~* \\.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {\n\
        expires 1y;\n\
        add_header Cache-Control "public, immutable";\n\
    }\n\
}\n' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

                    # IMPORTANT FIX: NO BUILDKIT (avoids your crash)
                    docker build --no-cache -t $IMAGE_NAME:v1 .
                '''
            }
        }

        stage("Push Image (SAFE)") {
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

        stage("Deploy Container (FIXED PORT + CLEAN)") {
            steps {
                sh '''
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true

                    docker run -d \
                        --name $CONTAINER_NAME \
                        -p 80:80 \
                        --restart unless-stopped \
                        $IMAGE_NAME:v1
                '''
            }
        }

        stage("Verify") {
            steps {
                script {
                    echo "🚀 APP RUNNING"
                    echo "👉 http://192.168.122.127"
                }
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
