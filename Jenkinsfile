pipeline {
    agent any
    
    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git Branch')
        string(name: 'DEPLOYMENT_ID', description: 'Unique Deployment ID')
    }
    
    environment {
        DOCKERHUB_USERNAME = credentials('dockerhub-username')
        DOCKERHUB_PASSWORD = credentials('dockerhub-password')
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                echo "Cloning repository: ${params.REPO_URL}"
                sh '''
                    rm -rf /tmp/${APP_NAME}
                    git clone --depth 1 -b ${BRANCH} ${REPO_URL} /tmp/${APP_NAME}
                '''
            }
        }
        
        stage('Detect Stack') {
            steps {
                script {
                    def stack = 'unknown'
                    if (fileExists('/tmp/${APP_NAME}/package.json')) {
                        stack = 'nodejs'
                    } else if (fileExists('/tmp/${APP_NAME}/requirements.txt')) {
                        stack = 'python'
                    } else if (fileExists('/tmp/${APP_NAME}/pom.xml')) {
                        stack = 'java'
                    } else if (fileExists('/tmp/${APP_NAME}/index.php')) {
                        stack = 'php'
                    } else if (fileExists('/tmp/${APP_NAME}/go.mod')) {
                        stack = 'go'
                    } else if (fileExists('/tmp/${APP_NAME}/Cargo.toml')) {
                        stack = 'rust'
                    } else if (fileExists('/tmp/${APP_NAME}/Gemfile')) {
                        stack = 'ruby'
                    }
                    env.STACK = stack
                    echo "Detected stack: ${stack}"
                }
            }
        }
        
        stage('Build Dockerfile') {
            steps {
                script {
                    if (fileExists('/tmp/${APP_NAME}/Dockerfile')) {
                        echo "Using existing Dockerfile"
                    } else {
                        echo "Auto-generating Dockerfile for ${env.STACK}"
                        sh """
                            cat > /tmp/${APP_NAME}/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
EOF
                        """
                    }
                }
            }
        }
        
        stage('Build Image') {
            steps {
                echo "Building Docker image..."
                sh '''
                    cd /tmp/${APP_NAME}
                    docker build -t ${APP_NAME}:${DEPLOYMENT_ID} .
                '''
            }
        }
        
        stage('Push to Registry') {
            steps {
                echo "Pushing to Docker Hub..."
                sh '''
                    echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin
                    docker tag ${APP_NAME}:${DEPLOYMENT_ID} ${DOCKERHUB_USERNAME}/${APP_NAME}:${DEPLOYMENT_ID}
                    docker push ${DOCKERHUB_USERNAME}/${APP_NAME}:${DEPLOYMENT_ID}
                '''
            }
        }
        
        stage('Deploy on VM') {
            steps {
                echo "Deploying container..."
                sh '''
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
                    docker pull ${DOCKERHUB_USERNAME}/${APP_NAME}:${DEPLOYMENT_ID}
                    docker run -d --name ${APP_NAME} \
                        --label "traefik.enable=true" \
                        --label "traefik.http.routers.${APP_NAME}.rule=Host(\`${APP_NAME}.localhost\`)" \
                        --label "traefik.http.routers.${APP_NAME}.entrypoints=web" \
                        --label "traefik.http.services.${APP_NAME}.loadbalancer.server.port=3000" \
                        ${DOCKERHUB_USERNAME}/${APP_NAME}:${DEPLOYMENT_ID}
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo "Waiting for container to start..."
                sh '''
                    sleep 5
                    docker logs ${APP_NAME}
                '''
            }
        }
    }
    
    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed. Cleaning up..."
            sh '''
                docker stop ${APP_NAME} || true
                docker rm ${APP_NAME} || true
            '''
        }
    }
}
