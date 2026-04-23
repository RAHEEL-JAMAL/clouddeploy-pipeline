pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL', defaultValue: '')
        string(name: 'APP_NAME', description: 'Application Name (letters/numbers only)', defaultValue: '')
        string(name: 'BRANCH', description: 'Optional branch (leave empty for auto)', defaultValue: '')
        string(name: 'DEPLOYMENT_ID', description: 'Version tag', defaultValue: 'v1')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
        SAFE_APP = ""
        BRANCH_NAME = ""
        IMAGE_NAME = ""
    }

    stages {

     stage('Validate Input') {
    steps {
        script {

            def app = (params.APP_NAME ?: "").toString().trim()

            echo "RAW APP_NAME: ${app}"

            if (!app) {
                error "APP_NAME is empty"
            }

            env.SAFE_APP = app.toLowerCase().replaceAll(/[^a-z0-9]/, "")

            echo "SANITIZED APP: ${env.SAFE_APP}"

            if (!env.SAFE_APP) {
                error "APP_NAME invalid after sanitization"
            }
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
                            script: "git ls-remote --heads ${params.REPO_URL}",
                            returnStdout: true
                        ).trim()

                        if (branches.contains("main")) {
                            env.BRANCH_NAME = "main"
                        } else if (branches.contains("master")) {
                            env.BRANCH_NAME = "master"
                        } else {
                            error "❌ No valid branch found (main/master missing)"
                        }
                    }

                    echo "✔ Using branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    sh "rm -rf /tmp/${env.SAFE_APP}"

                    sh """
                        git clone --depth 1 \
                        -b ${env.BRANCH_NAME} \
                        ${params.REPO_URL} \
                        /tmp/${env.SAFE_APP}
                    """
                }
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
                        error "❌ Unsupported project (no package.json or requirements.txt)"
                    }

                    echo "✔ STACK: ${env.STACK}"
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

                    env.IMAGE_NAME = "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${env.DEPLOYMENT_ID}"

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
                    echo "$DOCKER_CREDS_PSW" | docker login -u "$DOCKER_CREDS_USR" --password-stdin
                    docker push ${env.IMAGE_NAME}
                """
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker rm -f ${env.SAFE_APP} || true

                    docker run -d \
                    --name ${env.SAFE_APP} \
                    -p 3000:3000 \
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
            echo "🚀 Deployment SUCCESS"
        }

        failure {
            echo "❌ Deployment FAILED"
        }
    }
}
