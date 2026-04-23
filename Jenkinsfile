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
        DOCKERHUB_USERNAME = credentials('dockerhub-username')
        DOCKERHUB_PASSWORD = credentials('dockerhub-password')
    }

    stages {

        stage('Clone Repository') {
            steps {
                sh """
                    rm -rf /tmp/${params.APP_NAME}
                    git clone --depth 1 -b ${params.BRANCH} ${params.REPO_URL} /tmp/${params.APP_NAME}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {

                    def path = "/tmp/${params.APP_NAME}"
                    def stack = "unknown"

                    if (fileExists("${path}/package.json")) {
                        stack = "nodejs"
                    }
                    else if (fileExists("${path}/requirements.txt")) {
                        stack = "python"
                    }
                    else if (fileExists("${path}/pom.xml")) {
                        stack = "java"
                    }
                    else if (fileExists("${path}/go.mod")) {
                        stack = "go"
                    }
                    else if (fileExists("${path}/Dockerfile")) {
                        stack = "docker"
                    }

                    env.STACK = stack
                    echo "Detected Stack: ${stack}"
                }
            }
        }

        stage('Build Dockerfile (if needed)') {
            steps {
                script {

                    def path = "/tmp/${params.APP_NAME}"

                    if (!fileExists("${path}/Dockerfile")) {

                        if (env.STACK == "nodejs") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm","start"]
EOF
                            """
                        }

                        else if (env.STACK == "python") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM python:3.10-alpine
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python","app.py"]
EOF
                            """
                        }

                        else if (env.STACK == "java") {
                            sh """
cat > ${path}/Dockerfile <<EOF
FROM openjdk:17
WORKDIR /app
COPY . .
CMD ["java","-jar","app.jar"]
EOF
                            """
                        }

                        else {
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

                    env.IMAGE_NAME = "${env.DOCKERHUB_USERNAME}/${params.APP_NAME}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${params.APP_NAME}
                        docker build -t ${env.IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh """
                    echo ${DOCKERHUB_PASSWORD} | docker login -u ${DOCKERHUB_USERNAME} --password-stdin
                    docker push ${env.IMAGE_NAME}
                """
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

            script {
                sh "docker logs ${params.APP_NAME} || true"
            }
        }
    }
}
