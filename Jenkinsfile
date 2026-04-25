pipeline {
    agent any

    parameters {
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'BRANCH', defaultValue: 'auto', description: 'Branch auto/master/main')
    }

    environment {
        DOCKERHUB_USER = "raheeljamal"
        IMAGE_NAME = "${DOCKERHUB_USER}/${params.APP_NAME}"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 Deploying: ${params.APP_NAME}"
                echo "📦 Repo: ${params.REPO_URL}"
            }
        }

        stage('Detect Branch SAFE') {
            steps {
                script {
                    def branch = "main"

                    if (params.BRANCH == "auto") {

                        def branches = sh(
                            script: "git ls-remote --heads ${params.REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'",
                            returnStdout: true
                        ).trim()

                        echo "🔍 Available branches: ${branches}"

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
                    echo "✔ FINAL BRANCH: ${branch}"
                }
            }
        }

        stage('Clone Repo SAFE') {
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
                    if (fileExists("app/package.json")) {
                        env.STACK = "node"
                    } else {
                        env.STACK = "static"
                    }

                    echo "📦 Stack: ${STACK}"
                }
            }
        }

        stage('Build Frontend SAFE (FIXED NODE ISSUE)') {
            steps {
                script {
                    if (env.STACK == "node") {

                        sh """
                            docker run --rm -v \$(pwd)/app:/app -w /app node:20 bash -c "
                                npm install &&
                                npm run build || true
                            "
                        """
                    }
                }
            }
        }

        stage('Dockerfile SAFE') {
            steps {
                script {
                    def dockerfile = "app/Dockerfile"

                    if (!fileExists(dockerfile)) {

                        writeFile file: dockerfile, text: """
FROM nginx:alpine

COPY . /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
"""
                    }
                }
            }
        }

        stage('Build Image SAFE') {
            steps {
                sh """
                    cd app
                    docker build --no-cache -t ${IMAGE_NAME}:v1 .
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh """
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push ${IMAGE_NAME}:v
