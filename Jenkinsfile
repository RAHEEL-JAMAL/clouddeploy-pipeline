pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git Branch')
        string(name: 'DEPLOYMENT_ID', description: 'Unique Deployment ID')
        choice(name: 'DEPLOY_TARGET', choices: ['VM', 'AWS'], description: 'Deployment Target')
    }

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub-cred')
    }

    stages {

        stage('Prepare') {
            steps {
                script {
                    // ✅ sanitize app name (fix spaces + symbols)
                    def cleanName = params.APP_NAME
                        .toLowerCase()
                        .replaceAll("[^a-z0-9-]", "-")

                    env.APP_SAFE = cleanName

                    echo "Sanitized App Name: ${env.APP_SAFE}"
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    sh "rm -rf /tmp/${env.APP_SAFE}"

                    // ✅ auto detect default branch
                    def branch = sh(
                        script: "git ls-remote --symref ${params.REPO_URL} HEAD | grep -o 'refs/heads/.*' | sed 's#refs/heads/##'",
                        returnStdout: true
                    ).trim()

                    if (!branch) {
                        branch = params.BRANCH
                    }

                    echo "Using branch: ${branch}"

                    sh """
                        git clone --depth 1 -b ${branch} ${params.REPO_URL} /tmp/${env.APP_SAFE}
                    """
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def path = "/tmp/${env.APP_SAFE}"
                    def stack = "unknown"

                    if (fileExists("${path}/package.json")) {
                        stack = "nodejs"
                    } else if (fileExists("${path}/requirements.txt")) {
                        stack = "python"
                    } else if (fileExists("${path}/pom.xml")) {
                        stack = "java"
                    } else if (fileExists("${path}/Dockerfile")) {
                        stack = "docker"
                    }

                    env.STACK = stack
                    echo "Detected Stack: ${stack}"
                }
            }
        }

        stage('Build Dockerfile') {
            steps {
                script {
                    def path = "/tmp/${env.APP_SAFE}"

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
                        } else if (env.STACK == "python") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM python:3.10-alpine
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python","app.py"]
EOF
                            """
                        } else if (env.STACK == "java") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM openjdk:17
WORKDIR /app
COPY . .
CMD ["java","-jar","app.jar"]
EOF
                            """
                        } else {
                            error "Unsupported project type"
                        }

                    } else {
                        echo "Using existing Dockerfile"
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    def IMAGE = "${DOCKERHUB_CREDS_USR}/${env.APP_SAFE}:${params.DEPLOYMENT_ID}"
                    env.IMAGE_NAME = IMAGE

                    sh """
                        cd /tmp/${env.APP_SAFE}
                        docker build -t ${IMAGE} .
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    sh """
                        echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin
                        docker push ${env.IMAGE_NAME}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {

                    def port = "3000"

                    if (params.DEPLOY_TARGET == "VM") {

                        sh """
docker rm -f ${env.APP_SAFE} || true
docker pull ${env.IMAGE_NAME}

docker run -d -p ${port}:3000 --name ${env.APP_SAFE} ${env.IMAGE_NAME}
                        """

                        echo "App URL: http://YOUR_VM_IP:${port}"

                    } else {

                        sh """
ssh -o StrictHostKeyChecking=no ubuntu@EC2_IP '
docker rm -f ${env.APP_SAFE} || true
docker pull ${env.IMAGE_NAME}
docker run -d -p 80:3000 --name ${env.APP_SAFE} ${env.IMAGE_NAME}
'
                        """
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh """
                    sleep 5
                    docker ps | grep ${env.APP_SAFE} || true
                """
            }
        }
    }

    post {
        success {
            echo "Deployment SUCCESS 🚀"
        }

        failure {
            echo "Deployment FAILED ❌"
            sh "docker logs ${env.APP_SAFE} || true"
        }
    }
}
