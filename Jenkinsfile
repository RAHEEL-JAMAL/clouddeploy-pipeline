]pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
        string(name: 'APP_NAME', defaultValue: 'myapp123', description: 'App name (letters/numbers only)')
        string(name: 'BRANCH', defaultValue: '', description: 'Branch (optional)')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Version tag')
    }

    environment {
        SAFE_APP = ""
        BRANCH_NAME = ""
        IMAGE_NAME = ""
    }

    stages {

        stage('Validate Input') {
            steps {
                script {

                    echo "RAW APP_NAME: ${params.APP_NAME}"

                    // FIX 1: correct sanitization (IMPORTANT)
                    def clean = params.APP_NAME?.toString().trim()
                                    .replaceAll(/[^a-zA-Z0-9]/, "")
                                    .toLowerCase()

                    if (!clean || clean.length() == 0) {
                        error "❌ APP_NAME invalid after sanitization (use only letters/numbers)"
                    }

                    env.SAFE_APP = clean
                    env.DEPLOYMENT_ID = params.DEPLOYMENT_ID ?: "v1"

                    echo "✔ SAFE APP: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {

                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH.trim()
                    } else {

                        // FIX 2: proper default branch detection
                        def branch = sh(
                            script: """
                                git ls-remote --heads ${params.REPO_URL} \
                                | awk '{print \$2}' \
                                | head -n 1 \
                                | sed 's#refs/heads/##'
                            """,
                            returnStdout: true
                        ).trim()

                        env.BRANCH_NAME = branch ?: "main"
                    }

                    echo "✔ Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    sh """
                        rm -rf /tmp/${env.SAFE_APP}
                        git clone --depth 1 -b ${env.BRANCH_NAME} ${params.REPO_URL} /tmp/${env.SAFE_APP}
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
                        error "❌ Unsupported project type"
                    }

                    echo "STACK: ${env.STACK}"
                }
            }
        }

        stage('Build Image') {
            steps {
                script {

                    env.IMAGE_NAME = "raheeljamal/${env.SAFE_APP}:${env.DEPLOYMENT_ID}"

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
                    echo \$DOCKER_CREDS_PSW | docker login -u \$DOCKER_CREDS_USR --password-stdin
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
            echo "❌ Deployment FAILED (check logs above)"
        }
    }
}
