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

        stage('Clone Repository') {
            steps {
                script {
                    sh "rm -rf /tmp/${params.APP_NAME}"

                    // AUTO FIX branch (main → master)
                    def branchExists = sh(
                        script: "git ls-remote --heads ${params.REPO_URL} ${params.BRANCH}",
                        returnStatus: true
                    )

                    def finalBranch = params.BRANCH

                    if (branchExists != 0) {
                        echo "Branch '${params.BRANCH}' not found → using master"
                        finalBranch = "master"
                    }

                    echo "Using branch: ${finalBranch}"

                    sh """
                        git clone --depth 1 -b ${finalBranch} ${params.REPO_URL} /tmp/${params.APP_NAME}
                    """
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    def path = "/tmp/${params.APP_NAME}"
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
                    def path = "/tmp/${params.APP_NAME}"

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

                    // ✅ FIX: define IMAGE_NAME HERE (not environment block)
                    def IMAGE_NAME = "${DOCKERHUB_CREDS_USR}/${params.APP_NAME}:${params.DEPLOYMENT_ID}"

                    env.IMAGE_NAME = IMAGE_NAME

                    sh """
                        cd /tmp/${params.APP_NAME}
                        docker build -t ${IMAGE_NAME} .
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
docker stop ${params.APP_NAME} || true
docker rm ${params.APP_NAME} || true

docker pull ${env.IMAGE_NAME}

docker run -d -p ${port}:3000 --name ${params.APP_NAME} ${env.IMAGE_NAME}
                        """

                    } else {

                        sh """
ssh -o StrictHostKeyChecking=no ubuntu@EC2_IP '
docker stop ${params.APP_NAME} || true
docker rm ${params.APP_NAME} || true

docker pull ${env.IMAGE_NAME}

docker run -d -p 80:3000 --name ${params.APP_NAME} ${env.IMAGE_NAME}
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
                    docker ps | grep ${params.APP_NAME} || true
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
            sh "docker logs ${params.APP_NAME} || true"
        }
    }
}
