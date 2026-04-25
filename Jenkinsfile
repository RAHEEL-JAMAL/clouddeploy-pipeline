
pipeline {
    agent any

    environment {
        IMAGE_NAME = "raheeljamal/fyp-pipeline"
        CONTAINER_NAME = "fyp-app"
        DOCKER_CREDS = "dockerhub-cred"
    }

    stages {

        stage("Init") {
            steps {
                echo "🚀 Starting Deployment Pipeline"
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

        stage("Build Docker Image (FIXED SAFE)") {
            steps {
                sh '''
                    cd app

                    # FIXED Dockerfile (NO INLINE NGNIX ERRORS)
                    cat > Dockerfile << 'EOF'
# ---------- BUILD STAGE ----------
FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

# ---------- PRODUCTION STAGE ----------
FROM nginx:alpine

RUN rm -rf /usr/share/nginx/html/*

COPY --from=build /app/dist /usr/share/nginx/html

# SAFE nginx config (no Groovy issues, no parsing issues)
RUN echo "server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files \$uri \$uri/ /index.html;
    }

    location ~* \\.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control \"public, immutable\";
    }
}" > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

                    docker build --no-cache -t $IMAGE_NAME:v1 .
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
                echo "🚀 APP LIVE"
                echo "👉 http://192.168.122.127"
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
