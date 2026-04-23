pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')
        string(name: 'BRANCH', defaultValue: '', description: 'Leave empty for auto-detect')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Deployment ID')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
        SAFE_APP = ""
        BRANCH_NAME = ""
        IMAGE_NAME = ""
    }

    stages {

        stage('Prepare') {
            steps {
                script {

                    if (!params.REPO_URL?.trim()) {
                        error "REPO_URL is required ❌"
                    }

                    if (!params.APP_NAME?.trim()) {
                        error "APP_NAME is required ❌"
                    }

                    // SAFE APP NAME (bulletproof)
                    env.SAFE_APP = params.APP_NAME
                        .toLowerCase()
                        .replaceAll(/[^a-z0-9]/, "")

                    if (!env.SAFE_APP || env.SAFE_APP.trim() == "") {
                        error "APP_NAME invalid after sanitization ❌ (use letters/numbers only)"
                    }

                    env.DEPLOYMENT_ID = params.DEPLOYMENT_ID ?: "v1"

                    echo "SAFE APP NAME: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {

                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH
                    } else {

                        def branches = sh(
                            script: """
                                git ls-remote --heads ${params.REPO_URL} | awk '{print \$2}' | sed 's#refs/heads/##'
                            """,
                            returnStdout: true
                        ).trim()

                        if (branches.contains("main")) {
                            env.BRANCH_NAME = "main"
                        } else if (branches.contains("master")) {
                            env.BRANCH_NAME = "master"
                        } else {
                            env.BRANCH_NAME = branches.split("\\n")[0]
                        }
                    }

                    echo "Using branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repository') {
            steps {
                sh """
                    rm -rf /tmp/${env.SAFE_APP}

                    git clone --depth 1 -b ${env.BRANCH_NAME} \
                    ${params.REPO_URL} \
                    /tmp/${env.SAFE_APP}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def path = "/tmp/${env.SAFE_APP}"

                    if (fileExists("${path}/package.json")) {
                        env.STACK = "nodejs"
                    } else if (fileExists("${path}/requirements.txt")) {
                        env.STACK = "python"
                    } else {
                        error "Unsupported project type ❌"
                    }

                    echo "STACK: ${env.STACK}"
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    def path = "/tmp/${env.SAFE_APP}"

                    if (!fileExists("${path}/Dockerfile")) {

                        if (env.STACK == "nodejs") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm","start"]
EOF
                            """
                        }

                        else if (env.STACK == "python") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python","app.py"]
EOF
                            """
                        }
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {

                    env.IMAGE_NAME =
                        "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${env.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${env.SAFE_APP}
                        docker build -t ${env.IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Push Image') {
            steps {
                sh """
                    echo "${DOCKER_CREDS_PSW}" | docker login -u "${DOCKER_CREDS_USR}" --password-stdin
                    docker push ${env.IMAGE_NAME}
                """
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker stop ${env.SAFE_APP} || true
                    docker rm ${env.SAFE_APP} || true

                    docker run -d -p 3000:3000 \
                    --name ${env.SAFE_APP} \
                    ${env.IMAGE_NAME}
                """
            }
        }

        stage('Verify') {
            steps {
                sh "docker ps | grep ${env.SAFE_APP} || true"
            }
        }
    }

    post {
        success {
            echo "Deployment SUCCESS 🚀"
        }

        failure {
            echo "Deployment FAILED ❌"
            sh "docker logs ${env.SAFE_APP} || true"
        }
    }
}
