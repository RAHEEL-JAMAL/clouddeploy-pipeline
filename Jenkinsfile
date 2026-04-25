pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "raheeljamal"
        IMAGE_NAME     = "${DOCKERHUB_USER}/${APP_NAME}"
        DOCKER_BUILDKIT = "1"
    }

    parameters {
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'BRANCH', defaultValue: 'auto', description: 'Branch auto/master/main')
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 Deploying: ${params.APP_NAME}"
                echo "📦 Repo: ${params.REPO_URL}"
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    def branch = "main"

                    if (params.BRANCH == "auto") {
                        def branches = sh(
                            script: """git ls-remote --heads ${params.REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'""",
                            returnStdout: true
                        ).trim()

                        echo "🔍 Branches: ${branches}"

                        if (branches.contains("main")) {
                            branch = "main"
                        } else if (branches.contains("master")) {
                            branch = "master"
                        } else {
                            branch = branches.tokenize("\n")[0]
                        }
                    } else {
                        branch = params.BRANCH
                    }

                    env.BRANCH = branch
                    echo "✔ Branch: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh """
                    rm -rf app
                    git clone --depth 1 -b ${BRANCH} ${REPO_URL} app
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("app/manage.py")) {
                        env.STACK = "django"
                    } else if (fileExists("app/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "static"
                    }

                    echo "📦 Stack: ${env.STACK}"
                }
            }
        }

        stage('Build App (if needed)') {
            steps {
                script {
                    if (env.STACK == "node") {
                        sh '''
                            docker run --rm \
                              -v "$PWD/app:/app" \
                              -w /app \
                              node:20-alpine \
                              sh -c "npm install && npm run build || true"
                        '''
                        echo "✅ Node build done"
                    } else {
                        echo "ℹ️ No build required"
                    }
                }
            }
        }

        stage('Prepare Dockerfile') {
            steps {
                script {
                    if (!fileExists("app/Dockerfile")) {

                        if (env.STACK == "django") {
                            writeFile file: "app/Dockerfile", text: '''
FROM python:3.9
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
'''
                        } else {
                            writeFile file: "app/Dockerfile", text: '''
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
                        }

                        echo "📝 Dockerfile created"
                    } else {
                        echo "📝 Using existing Dockerfile"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh """
                    cd app
                    docker build -t ${IMAGE_NAME}:v1 .
                """
            }
        }

        stage('Push Image (RETRY FIXED)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh '''
                        set -e

                        echo "🔐 Logging into DockerHub..."

                        for i in 1 2 3; do
                            echo "$PASS" | docker login -u "$USER" --password-stdin && break
                            echo "⚠️ Login failed retry $i"
                            sleep 5
                        done

                        echo "🚀 Pushing image with retry..."

                        for i in 1 2 3; do
                            docker push ${IMAGE_NAME}:v1 && break
                            echo "⚠️ Push failed retry $i"
                            sleep 10
                        done
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {

                    def hostPort = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    def containerPort = "80"
                    if (env.STACK == "django") {
                        containerPort = "8000"
                    }

                    echo "🧠 Mapping: ${hostPort} -> ${containerPort}"

                    sh """
                        docker stop ${params.APP_NAME} || true
                        docker rm ${params.APP_NAME} || true

                        docker run -d \
                        -p ${hostPort}:${containerPort} \
                        --name ${params.APP_NAME} \
                        ${IMAGE_NAME}:v1
                    """

                    echo "🌐 LIVE: http://192.168.122.127:${hostPort}"
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
            echo "🚀 SUCCESS"
        }
        failure {
            echo "❌ FAILED"
        }
    }
}
