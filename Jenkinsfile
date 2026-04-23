pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')

        // default is now empty → we auto-detect main/master
        string(name: 'BRANCH', defaultValue: '', description: 'Git Branch (optional: auto main/master)')

        string(name: 'DEPLOYMENT_ID', description: 'Unique Deployment ID')
        choice(name: 'DEPLOY_TARGET', choices: ['VM', 'AWS'], description: 'Deployment Target')
    }

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub-cred')
    }

    stages {

        stage('Clone Repository (Auto Branch Fix)') {
            steps {
                script {

                    def repoDir = "/tmp/${params.APP_NAME}"

                    sh "rm -rf ${repoDir}"

                    // 🔥 AUTO BRANCH LOGIC
                    def branch = params.BRANCH

                    if (branch == null || branch.trim() == "") {
                        echo "No branch provided → trying main/master auto detection"

                        sh """
                            git ls-remote --heads ${params.REPO_URL} > /tmp/branches.txt
                        """

                        def branches = readFile('/tmp/branches.txt')

                        if (branches.contains("refs/heads/main")) {
                            branch = "main"
                        } else if (branches.contains("refs/heads/master")) {
                            branch = "master"
                        } else {
                            error "No main or master branch found in repo"
                        }

                        echo "Auto-selected branch: ${branch}"
                    }

                    env.SELECTED_BRANCH = branch

                    sh """
                        git clone --depth 1 -b ${branch} ${params.REPO_URL} ${repoDir}
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
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                script {

                    env.IMAGE_NAME = "${DOCKERHUB_CREDS_USR}/${params.APP_NAME}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${params.APP_NAME}
                        docker build -t ${env.IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                sh '''
                    echo "$DOCKERHUB_CREDS_PSW" | docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin
                    docker push $IMAGE_NAME
                '''
            }
        }

        stage('Deploy') {
            steps {
                script {

                    if (params.DEPLOY_TARGET == "VM") {

                        sh '''
docker stop $APP_NAME || true
docker rm $APP_NAME || true

docker run -d -p 3000:3000 --name $APP_NAME $IMAGE_NAME
'''
                    } else {

                        sh '''
ssh -o StrictHostKeyChecking=no ubuntu@EC2_IP "
docker stop $APP_NAME || true
docker rm $APP_NAME || true

docker pull $IMAGE_NAME
docker run -d -p 80:3000 --name $APP_NAME $IMAGE_NAME
"
'''
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    docker ps | grep $APP_NAME || true
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment SUCCESS 🚀"
        }

        failure {
            echo "Deployment FAILED ❌"
        }
    }
}
