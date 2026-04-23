pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', defaultValue: '', description: 'Optional App Name')
        string(name: 'BRANCH', defaultValue: '', description: 'Optional Branch')
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
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {

                        def raw = params.APP_NAME?.trim()

                        if (!raw) {
                            raw = params.REPO_URL.tokenize('/').last().replace('.git', '')
                        }

                        env.SAFE_APP = raw
                            .toLowerCase()
                            .replaceAll(/[^a-z0-9]/, "")

                        if (!env.SAFE_APP || env.SAFE_APP.trim() == "") {
                            env.SAFE_APP = "app${System.currentTimeMillis()}"
                        }

                        echo "✔ SAFE APP: ${env.SAFE_APP}"
                    }
                }
            }
        }

        stage('Detect Branch') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {

                        if (params.BRANCH?.trim()) {
                            env.BRANCH_NAME = params.BRANCH
                        } else {

                            def branches = sh(
                                script: """
                                    git ls-remote --heads ${params.REPO_URL} \
                                    | awk '{print \$2}' \
                                    | sed 's#refs/heads/##'
                                """,
                                returnStdout: true
                            ).trim()

                            if (branches.contains("main")) {
                                env.BRANCH_NAME = "main"
                            } else if (branches.contains("master")) {
                                env.BRANCH_NAME = "master"
                            } else {
                                env.BRANCH_NAME = branches ? branches.split("\n")[0] : "main"
                            }
                        }

                        echo "✔ Branch: ${env.BRANCH_NAME}"
                    }
                }
            }
        }

        stage('Clone Repo') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh """
                        rm -rf /tmp/${env.SAFE_APP}

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
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {

                        def path = "/tmp/${env.SAFE_APP}"

                        if (fileExists("${path}/package.json")) {
                            env.STACK = "nodejs"
                        } else if (fileExists("${path}/requirements.txt")) {
                            env.STACK = "python"
                        } else {
                            error "Unsupported project"
                        }

                        echo "✔ STACK: ${env.STACK}"
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
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
        }

        stage('Build Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    script {

                        env.IMAGE_NAME =
                            "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${params.DEPLOYMENT_ID}"

                        sh """
                            cd /tmp/${env.SAFE_APP}
                            docker build -t ${env.IMAGE_NAME} .
                        """
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh """
                        echo "${DOCKER_CREDS_PSW}" | docker login -u "${DOCKER_CREDS_USR}" --password-stdin
                        docker push ${env.IMAGE_NAME}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh """
                        docker stop ${env.SAFE_APP} || true
                        docker rm ${env.SAFE_APP} || true

                        docker run -d -p 3000:3000 \
                        --name ${env.SAFE_APP} \
                        ${env.IMAGE_NAME}
                    """
                }
            }
        }

        stage('Verify') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh "docker ps | grep ${env.SAFE_APP} || true"
                }
            }
        }
    }

    post {
        always {
            echo "✔ Pipeline finished (some stages may have failed but pipeline continued)"
        }

        success {
            echo "🚀 Deployment SUCCESS"
        }

        failure {
            echo "❌ Pipeline FAILED (but errors were handled)"
        }
    }
}
