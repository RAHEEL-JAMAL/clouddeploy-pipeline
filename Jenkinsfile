pipeline {
    agent any

    environment {
        IMAGE_NAME = "raheeljamal/erererrerere"
        IMAGE_TAG = "v1"
        REPO_URL = "https://github.com/RAHEEL-JAMAL/LondheShubham153-django-notes-app.git"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 Starting Deployment Pipeline"
                echo "📦 Repo: ${REPO_URL}"
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branches = sh(
                        script: "git ls-remote --heads ${REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim()

                    echo "🔍 Branches: ${branches}"

                    env.BRANCH = "main"
                    echo "✔ Branch selected: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh '''
                    rm -rf app
                    git clone --depth 1 -b main ${REPO_URL} app
                '''
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists('app/manage.py')) {
                        echo "📦 Stack: Django"
                        env.STACK = "django"
                    } else if (fileExists('app/package.json')) {
                        echo "📦 Stack: NodeJS"
                        env.STACK = "node"
                    } else {
                        echo "📦 Stack: Unknown"
                        env.STACK = "unknown"
                    }
                }
            }
        }

        stage('Build App (if needed)') {
            steps {
                script {
                    if (env.STACK == "node") {
                        sh '''
                            cd app
                            npm install
                            npm run build || echo "No build step"
                        '''
                    } else {
                        echo "ℹ️ No build required"
                    }
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    if (!fileExists('app/Dockerfile')) {
                        echo "📝 No Dockerfile found - using default logic"
                    } else {
                        echo "📝 Using existing Dockerfile"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    cd app

                    echo "🛠 Disabling BuildKit to fix buildx error"
                    export DOCKER_BUILDKIT=0

                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "📤 Logging into Docker Hub"
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "🚀 Deploying container..."

                    docker stop app_container || true
                    docker rm app_container || true

                    docker run -d \
                        --name app_container \
                        -p 8000:8000 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "🔍 Checking running containers..."
                    docker ps
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment SUCCESS"
        }
        failure {
            echo "❌ Deployment FAILED"
        }
    }
}
