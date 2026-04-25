pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
    }

    stages {

        stage('Init') {
            steps {
                echo "🚀 MULTI APP SAFE DEPLOY STARTED"
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    env.REPO_URL = params.REPO_URL
                    env.APP_ID = sh(script: "echo ${REPO_URL} | md5sum | cut -c1-6", returnStdout: true).trim()

                    env.CONTAINER_NAME = "app_${APP_ID}"

                    echo "📦 Repo: ${REPO_URL}"
                    echo "🆔 App ID: ${APP_ID}"
                }
            }
        }

        stage('Allocate Safe Port') {
            steps {
                script {

                    // get all used ports
                    def usedPorts = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]\\+->' | cut -d'>' -f1 || true",
                        returnStdout: true
                    ).trim()

                    def port = 3000

                    while (usedPorts.contains(port.toString())) {
                        port = port + 1
                    }

                    env.EXTERNAL_PORT = port.toString()

                    echo "🌐 Assigned Safe Port: ${EXTERNAL_PORT}"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                sh '''
                    rm -rf app
                    git clone --depth 1 ${REPO_URL} app
                '''
            }
        }

        stage('Detect Stack') {
            steps {
                script {

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                        env.INTERNAL_PORT = "8000"

                    } else if (fileExists('app/package.json')) {
                        env.STACK = "node"
                        env.INTERNAL_PORT = "80"

                    } else if (fileExists('app/index.php')) {
                        env.STACK = "php"
                        env.INTERNAL_PORT = "80"

                    } else {
                        env.STACK = "unknown"
                        env.INTERNAL_PORT = "80"
                    }

                    echo "📦 Stack: ${STACK}"
                    echo "🔌 Internal Port: ${INTERNAL_PORT}"
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    cd app
                    docker build -t ${IMAGE_NAME}:${APP_ID} .
                '''
            }
        }

        stage('Stop Only Same App') {
            steps {
                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''
            }
        }

        stage('Run Container (PARALLEL SAFE)') {
            steps {
                script {

                    if (env.STACK == "django") {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:8000 \
                            ${IMAGE_NAME}:${APP_ID}
                        """
                    }

                    else {
                        sh """
                            docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${EXTERNAL_PORT}:80 \
                            ${IMAGE_NAME}:${APP_ID}
                        """
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh 'docker ps'
            }
        }
    }

    post {
        success {
            echo "✅ DEPLOY SUCCESS"
            echo "🌐 App URL → http://VM_IP:${EXTERNAL_PORT}"
        }

        failure {
            echo "❌ DEPLOY FAILED"
        }
    }
}
