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
                    echo "✔ Using branch: ${branch}"
                }
            }
        }

        stage('Clone Repo SAFE') {
            steps {
                script {
                    sh """
                        rm -rf app
                        git clone --depth 1 -b ${BRANCH} ${REPO_URL} app
                    """
                }
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

        stage('Build Frontend SAFE') {
            steps {
                script {
                    if (env.STACK == "node") {
                        sh """
                            cd app
                            npm install
                            npm run build
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

        stage('Build Image') {
            steps {
                sh """
                    cd app
                    docker build -t ${IMAGE_NAME}:v1 .
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
                        docker push ${IMAGE_NAME}:v1
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def port = sh(script: "shuf -i 3000-3999 -n 1", returnStdout: true).trim()

                    sh """
                        docker stop ${params.APP_NAME} || true
                        docker rm ${params.APP_NAME} || true
                        docker run -d -p ${port}:80 --name ${params.APP_NAME} ${IMAGE_NAME}:v1
                    """

                    echo "🌐 LIVE: http://192.168.122.127:${port}"
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
