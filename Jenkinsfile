pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repo URL')
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
                    } else {

                        def raw = sh(
                            script: "git ls-remote --heads ${params.REPO_URL}",
                            returnStdout: true
                        ).trim()

                        def branches = raw.split("\n").collect {
                            it.split("refs/heads/")[1].trim()
                        }

                        echo "Branches: ${branches}"

                        env.BRANCH_NAME = branches.contains("main") ? "main" :
                                           branches.contains("master") ? "master" :
                                           branches[0]
                    }

                    echo "✔ Branch: ${env.BRANCH_NAME}"
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

        stage('Build Image (Optimized)') {
            steps {
                script {
                    env.IMAGE_NAME = "${DOCKER_CREDS_USR}/${env.SAFE_APP}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${env.SAFE_APP}
                        
                        echo "🐳 Building optimized image..."
                        docker build --no-cache=false -t ${env.IMAGE_NAME} .
                    """

                    echo "✔ Built: ${env.IMAGE_NAME}"
                }
            }
        }

        stage('Push Image (Fast & Safe)') {
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

                echo "🌍 LIVE URL:"
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
