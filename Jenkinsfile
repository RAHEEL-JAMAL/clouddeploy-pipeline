pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repository URL (required)')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name (letters/numbers only)')
        string(name: 'BRANCH', defaultValue: '', description: 'Optional branch (auto-detect if empty)')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Image tag')
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
                    echo "RAW APP_NAME: ${params.APP_NAME}"

                    if (!params.REPO_URL?.trim()) {
                        error "❌ REPO_URL is required"
                    }

                    def rawApp = (params.APP_NAME ?: "myapp").trim()

                    // SAFE sanitization
                    env.SAFE_APP = rawApp.replaceAll(/[^a-zA-Z0-9]/, "").toLowerCase()

                    if (!env.SAFE_APP?.trim()) {
                        env.SAFE_APP = "myapp"
                    }

                    echo "✔ SAFE APP NAME: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {
                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH
                    } else {
                        def result = sh(
                            script: "git ls-remote --heads ${params.REPO_URL}",
                            returnStdout: true
                        ).trim()

                        def branch = result.readLines()
                            .find { it.contains("refs/heads/") }
                            ?.split("refs/heads/")[1]

                        env.BRANCH_NAME = branch ?: "main"
                    }

                    echo "✔ Using branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
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
                script {
                    def path = "/tmp/${env.SAFE_APP}"

                    if (fileExists("${path}/package.json")) {
                        env.STACK = "nodejs"
                    } else if (fileExists("${path}/requirements.txt")) {
                        env.STACK = "python"
                    } else {
                        env.STACK = "unknown"
                    }

                    echo "STACK: ${env.STACK}"
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
                script {
                    sh """
                        echo "$DOCKER_CREDS_PSW" | docker login -u "$DOCKER_CREDS_USR" --password-stdin
                        docker push ${env.IMAGE_NAME}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
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
