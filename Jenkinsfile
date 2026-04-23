pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', description: 'Git Repository URL')
        string(name: 'APP_NAME', description: 'Application Name')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Preferred Branch')
        string(name: 'DEPLOYMENT_ID', description: 'Deployment ID')
        choice(name: 'DEPLOY_TARGET', choices: ['VM', 'AWS'], description: 'Deploy Target')
    }

    environment {
        IMAGE_NAME = ""
    }

    stages {

        stage('Clone Repository') {
            steps {
                script {
                    def path = "/tmp/${params.APP_NAME}"
                    sh "rm -rf ${path}"

                    def branchToUse = params.BRANCH

                    def result = sh(
                        script: "git ls-remote --heads ${params.REPO_URL} ${params.BRANCH}",
                        returnStdout: true
                    ).trim()

                    if (result == "") {
                        echo "Branch '${params.BRANCH}' not found → using master"
                        branchToUse = "master"
                    }

                    echo "Using branch: ${branchToUse}"

                    sh """
                        git clone --depth 1 -b ${branchToUse} ${params.REPO_URL} ${path}
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
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm","start"]
EOF
"""
                        } else {
                            error "Unsupported project"
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
                    env.IMAGE_NAME = "${params.APP_NAME}:${params.DEPLOYMENT_ID}"

                    sh """
                        cd /tmp/${params.APP_NAME}
                        docker build -t ${env.IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker tag ${env.IMAGE_NAME} $USER/${params.APP_NAME}:${params.DEPLOYMENT_ID}
                        docker push $USER/${params.APP_NAME}:${params.DEPLOYMENT_ID}
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
docker stop ${params.APP_NAME} || true
docker rm ${params.APP_NAME} || true

docker run -d -p 3000:3000 --name ${params.APP_NAME} ${params.APP_NAME}:${params.DEPLOYMENT_ID}
                """
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
        }
    }
}
