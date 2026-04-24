pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repository URL')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'BRANCH', defaultValue: '', description: 'Optional branch')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Image tag')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.SAFE_APP = params.APP_NAME.replaceAll(/[^a-zA-Z0-9]/, "").toLowerCase()
                    if (!env.SAFE_APP?.trim()) {
                        env.SAFE_APP = "myapp"
                    }

                    echo "🚀 App: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {

                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH.trim()
                        echo "✔ Using user branch: ${env.BRANCH_NAME}"
                    } else {

                        def raw = sh(
                            script: "git ls-remote --heads ${params.REPO_URL}",
                            returnStdout: true
                        ).trim()

                        def branches = []
                        raw.split("\n").each { line ->
                            if (line.contains("refs/heads/")) {
                                branches.add(line.split("refs/heads/")[1].trim())
                            }
                        }

                        echo "Branches found: ${branches}"

                        if (branches.contains("main")) {
                            env.BRANCH_NAME = "main"
                        } else if (branches.contains("master")) {
                            env.BRANCH_NAME = "master"
                        } else {
                            env.BRANCH_NAME = branches[0]
                        }

                        echo "✔ Selected branch: ${env.BRANCH_NAME}"
                    }
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh """
                    rm -rf /tmp/${env.SAFE_APP}
                    git clone --depth 1 -b ${env.BRANCH_NAME} ${params.REPO_URL} /tmp/${env.SAFE_APP}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def path = "/tmp/${env.SAFE_APP}"

                    if (fileExists("${path}/package.json")) {
                        env.STACK = "node"
                    } else if (fileExists("${path}/requirements.txt")) {
                        env.STACK = "python"
                    } else {
                        env.STACK = "node"
                    }

                    echo "📦 Stack: ${env.STACK}"
                }
            }
        }

        stage('Ensure Dockerfile') {
            steps {
                script {
                    def path = "/tmp/${env.SAFE_APP}"

                    if (!fileExists("${path}/Dockerfile")) {

                        echo "⚠ No Dockerfile found → generating..."

                        if (env.STACK == "node") {
                            writeFile file: "${path}/Dockerfile", text: '''
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm","start"]
'''
                        } else {
                            writeFile file: "${path}/Dockerfile", text: '''
FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python","app.py"]
'''
                        }
                    } else {
                        echo "✔ Dockerfile exists"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    env.IMAGE_NAME = "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${env.SAFE_APP}
                        docker build -t ${env.IMAGE_NAME} .
                    """

                    echo "🐳 Built: ${env.IMAGE_NAME}"
                }
            }
        }

        stage('Push Image') {
            steps {
                sh """
                    echo "$DOCKER_CREDS_PSW" | docker login -u "$DOCKER_CREDS_USR" --password-stdin
                    docker push ${IMAGE_NAME}
                """
            }
        }

        stage('Deploy Container') {
            steps {
                script {
                    sh """
                        docker stop ${env.SAFE_APP} || true
                        docker rm ${env.SAFE_APP} || true

                        PORT=\$(shuf -i 3000-3999 -n 1)

                        docker run -d \
                        -p \$PORT:3000 \
                        --name ${env.SAFE_APP} \
                        ${env.IMAGE_NAME}

                        echo \$PORT > /tmp/${env.SAFE_APP}_port.txt
                    """

                    env.APP_PORT = sh(
                        script: "cat /tmp/${env.SAFE_APP}_port.txt",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Verify') {
            steps {
                sh "docker ps | grep ${env.SAFE_APP} || true"

                echo "🌍 App running at:"
                echo "👉 http://192.168.122.127:${env.APP_PORT}"
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
