pipeline {
    agent any

    parameters {
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'BRANCH', defaultValue: 'auto', description: 'auto / main / master')
    }

    environment {
        DOCKERHUB_USER = "raheeljamal"
        IMAGE_NAME = "${DOCKERHUB_USER}/${params.APP_NAME}"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 Deploying: ${params.APP_NAME}"
            }
        }

        stage('Detect Branch (FIXED SAFE)') {
            steps {
                script {

                    def branches = sh(
                        script: "git ls-remote --heads ${params.REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim()

                    echo "🔍 Available branches: ${branches}"

                    def selected = "master"

                    if (params.BRANCH != "auto") {
                        selected = params.BRANCH
                    } else {

                        if (branches.contains("main")) {
                            selected = "main"
                        } else if (branches.contains("master")) {
                            selected = "master"
                        } else {
                            selected = branches.tokenize("\n")[0]
                        }
                    }

                    env.BRANCH = selected
                    echo "✔ FINAL BRANCH: ${env.BRANCH}"
                }
            }
        }

        stage('Clone Repo (SAFE GUARANTEED)') {
            steps {
                script {
                    sh """
                        rm -rf app

                        # TRY selected branch
                        git clone --depth 1 -b ${BRANCH} ${REPO_URL} app || (

                            echo "⚠ Clone failed, trying fallback master..."

                            git clone --depth 1 -b master ${REPO_URL} app || (

                                echo "⚠ Master failed, trying default branch..."

                                git clone --depth 1 ${REPO_URL} app
                            )
                        )
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

        stage('Build (SAFE NODE)') {
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

        stage('Dockerfile FIXED') {
            steps {
                script {
                    if (!fileExists("app/Dockerfile")) {

                        writeFile file: "app/Dockerfile", text: """
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
                        docker stop ${APP_NAME} || true
                        docker rm ${APP_NAME} || true
                        docker run -d -p ${port}:80 --name ${APP_NAME} ${IMAGE_NAME}:v1
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
}
