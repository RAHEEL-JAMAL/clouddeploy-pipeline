pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')
        string(name: 'BRANCH', defaultValue: '', description: 'Leave empty for auto-detect')
        string(name: 'DEPLOYMENT_ID', description: 'Deployment ID')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
        IMAGE_NAME = ""
        SAFE_APP = ""
        BRANCH_NAME = ""
    }

    stages {

        stage('Prepare') {
            steps {
                script {
                    env.SAFE_APP = params.APP_NAME.replaceAll("[^a-zA-Z0-9]", "").toLowerCase()
                    echo "Sanitized Name: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch (FIXED)') {
            steps {
                script {
                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH
                    } else {
                        env.BRANCH_NAME = sh(
                            script: """
                            git ls-remote --symref ${params.REPO_URL} HEAD | grep refs/heads | head -n1 | sed 's#.*/##' | tr -d '\\n'
                            """,
                            returnStdout: true
                        ).trim()
                    }

                    echo "Using branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repo (FIXED)') {
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
                        env.STACK = "nodejs"
                    } else if (fileExists("${path}/requirements.txt")) {
                        env.STACK = "python"
                    } else {
                        error "Unsupported stack"
                    }

                    echo "Stack: ${env.STACK}"
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

                        if (env.STACK == "python") {
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
                    env.IMAGE_NAME = "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${env.SAFE_APP}
                        docker build -t ${env.IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "$DOCKER_CREDS_PSW" | docker login -u "$DOCKER_CREDS_USR" --password-stdin
                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker stop ${env.SAFE_APP} || true
                    docker rm ${env.SAFE_APP} || true

                    docker run -d -p 3000:3000 --name ${env.SAFE_APP} ${env.IMAGE_NAME}
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
        failure {
            sh "docker logs ${env.SAFE_APP} || true"
        }
    }
}
