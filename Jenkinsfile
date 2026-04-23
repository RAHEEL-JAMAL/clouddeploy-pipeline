pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repository URL (required)')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name (letters/numbers only)')
        string(name: 'BRANCH', defaultValue: '', description: 'Optional branch')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Image tag/version')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
    }

    stages {

        stage('Validate Input') {
            steps {
                script {

                    echo "RAW APP_NAME: ${params.APP_NAME}"

                    if (!params.REPO_URL?.trim()) {
                        error "REPO_URL is required"
                    }

                    def rawApp = (params.APP_NAME ?: "myapp").toString().trim()

                    // SAFE sanitization
                    def safeApp = rawApp.replaceAll(/[^a-zA-Z0-9]/, "").toLowerCase()

                    if (!safeApp?.trim()) {
                        safeApp = "myapp"
                    }

                    env.SAFE_APP = safeApp

                    // ensure never null
                    env.SAFE_APP = env.SAFE_APP ?: "myapp"

                    echo "SAFE APP NAME: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {

                    def branchName = "main"

                    try {
                        if (params.BRANCH?.trim()) {
                            branchName = params.BRANCH.trim()
                        } else {
                            def output = sh(
                                script: "git ls-remote --heads ${params.REPO_URL}",
                                returnStdout: true
                            ).trim()

                            def match = output.readLines().find { it.contains("refs/heads/") }

                            if (match) {
                                branchName = match.split("refs/heads/")[1]
                            }
                        }
                    } catch (e) {
                        echo "⚠ Branch detection failed → using main"
                        branchName = "main"
                    }

                    env.BRANCH_NAME = branchName ?: "main"

                    echo "Using branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    def dir = "/tmp/${env.SAFE_APP}"

                    sh """
                        rm -rf ${dir}
                        git clone --depth 1 -b ${env.BRANCH_NAME} ${params.REPO_URL} ${dir}
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
                        env.STACK = "nodejs"
                    }

                    echo "STACK: ${env.STACK}"
                }
            }
        }

        stage('Build Image') {
            steps {
                script {

                    def imageName = "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${params.DEPLOYMENT_ID}"

                    if (!imageName?.contains("/")) {
                        error "Invalid image name generated"
                    }

                    env.IMAGE_NAME = imageName

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

                    def container = env.SAFE_APP

                    sh """
                        docker stop ${container} || true
                        docker rm ${container} || true

                        docker run -d -p 3000:3000 \
                        --name ${container} \
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
