pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repository URL (required)')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name (letters/numbers only)')
        string(name: 'BRANCH', defaultValue: '', description: 'Branch (optional)')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Image tag/version')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
        SAFE_APP = "myapp"
        BRANCH_NAME = "main"
        IMAGE_NAME = ""
        APP_PORT = ""
    }

    stages {

        stage('Validate Input') {
            steps {
                script {
                    echo "RAW APP_NAME: ${params.APP_NAME}"

                    if (!params.REPO_URL?.trim()) {
                        error "❌ REPO_URL is required"
                    }

                    def rawApp = (params.APP_NAME ?: "myapp").toString().trim()
                    def sanitized = rawApp.replaceAll(/[^a-zA-Z0-9]/, "").toLowerCase()

                    if (!sanitized?.trim()) {
                        sanitized = "myapp"
                    }

                    env.SAFE_APP = sanitized
                    echo "✔ SAFE APP NAME: ${env.SAFE_APP}"
                }
            }
        }

       
       
stage('Detect Branch') {
    steps {
        script {

            def branches = sh(
                script: "git ls-remote --heads ${params.REPO_URL}",
                returnStdout: true
            ).trim()

            def branchList = branches.readLines().collect {
                it.split("refs/heads/")[1]
            }

            echo "Available branches: ${branchList}"

            // PRIORITY ORDER
            if (branchList.contains("main")) {
                env.BRANCH_NAME = "main"
            } 
            else if (branchList.contains("master")) {
                env.BRANCH_NAME = "master"
            } 
            else {
                env.BRANCH_NAME = branchList[0]
            }

            echo "✔ Auto Selected Branch: ${env.BRANCH_NAME}"
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
                        env.STACK = "nodejs"
                    }

                    echo "✔ STACK: ${env.STACK}"
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
                script {
                    sh '''
                        echo "$DOCKER_CREDS_PSW" | docker login -u "$DOCKER_CREDS_USR" --password-stdin
                        docker push ${IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh """
                        docker stop ${env.SAFE_APP} || true
                        docker rm ${env.SAFE_APP} || true

                        PORT=\$(shuf -i 3000-3999 -n 1)

                        docker run -d -p \$PORT:3000 \
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
                script {
                    sh "docker ps | grep ${env.SAFE_APP} || true"

                    echo "🌐 APP RUNNING AT:"
                    echo "👉 http://192.168.122.127:${env.APP_PORT}"
                }
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
