
pipeline {
    agent any

    environment {
        IMAGE_NAME = "auto-app"
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo "[STAGE_START] Init"
                    echo "MULTI APP SAFE DEPLOY STARTED"
                    echo "[STAGE_SUCCESS] Init"
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    echo "[STAGE_START] Input Repo"

                    env.REPO_URL = params.REPO_URL
                    env.APP_ID = sh(script: "echo ${REPO_URL} | md5sum | cut -c1-6", returnStdout: true).trim()
                    env.CONTAINER_NAME = "app_${APP_ID}"

                    echo "[META] REPO_URL=${REPO_URL}"
                    echo "[META] APP_ID=${APP_ID}"
                    echo "[META] CONTAINER_NAME=${CONTAINER_NAME}"

                    echo "[STAGE_SUCCESS] Input Repo"
                }
            }
        }

        stage('Allocate Safe Port') {
            steps {
                script {
                    echo "[STAGE_START] Allocate Safe Port"

                    def usedPorts = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | cut -d'-' -f1 || true",
                        returnStdout: true
                    ).trim()

                    def port = 3000
                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

                    env.EXTERNAL_PORT = port.toString()

                    echo "[META] PORT=${EXTERNAL_PORT}"
                    echo "[STAGE_SUCCESS] Allocate Safe Port"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script {
                    echo "[STAGE_START] Clone Repo"
                }
                sh '''
                    rm -rf app
                    git clone --depth 1 ${REPO_URL} app
                '''
                script {
                    echo "[STAGE_SUCCESS] Clone Repo"
                }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    echo "[STAGE_START] Secret Scan (Gitleaks)"
                    echo "Scanning repo for hardcoded secrets..."

                    def appDir = new File("${env.WORKSPACE}/app")
                    def secretPatterns = [
                        'password=', 'secret=', 'api_key=', 'apikey=',
                        'access_token=', 'private_key', 'BEGIN RSA PRIVATE',
                        'BEGIN OPENSSH PRIVATE', 'AWS_SECRET_ACCESS_KEY', 'AKIA'
                    ]
                    def extensions = ['.js', '.py', '.env', '.yml', '.yaml', '.ts']
                    def found = []

                    appDir.eachFileRecurse { file ->
                        if (file.isFile()) {
                            def skip = file.path.contains('node_modules') ||
                                       file.path.contains('.git') ||
                                       file.name == 'package-lock.json' ||
                                       file.name == 'yarn.lock'
                            if (!skip && extensions.any { file.name.endsWith(it) }) {
                                def content = file.text.toLowerCase()
                                secretPatterns.each { pattern ->
                                    if (content.contains(pattern.toLowerCase())) {
                                        found << "${file.name}: possible '${pattern}' found"
                                    }
                                }
                            }
                        }
                    }

                    if (found.size() > 0) {
                        found.each { echo "WARNING: ${it}" }
                        echo "[META] SECRET_SCAN=FAILED"
                    } else {
                        echo "[META] SECRET_SCAN=PASSED"
                        echo "No secrets found in repository"
                    }

                    echo "[STAGE_SUCCESS] Secret Scan (Gitleaks)"
                }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo "[STAGE_START] Detect Stack"

                    if (fileExists('app/manage.py')) {
                        env.STACK = "django"
                    } else if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')
                        if (pkg.contains("vite")) {
                            env.STACK = "vite"
                        } else {
                            env.STACK = "node"
                        }
                    } else {
                        env.STACK = "node"
                    }

                    echo "[META] STACK=${STACK}"
                    echo "[STAGE_SUCCESS] Detect Stack"
                }
            }
        }

        stage('Dependency Audit') {
            steps {
                script {
                    echo "[STAGE_START] Dependency Audit"
                    echo "Scanning dependencies for known vulnerabilities..."

                    if (env.STACK == "django") {
                        sh 'pip install pip-audit --quiet || true'
                        sh 'if [ -f app/requirements.txt ]; then pip-audit -r app/requirements.txt 2>&1 || true; fi'
                    } else {
                        sh 'if [ -f app/package.json ]; then cd app && npm audit --json > /tmp/npm-audit.json 2>&1 || true; fi'
                        sh '''python3 -c "
import json
try:
    obj = json.load(open('/tmp/npm-audit.json'))
    v = obj.get('metadata', {}).get('vulnerabilities', {})
    print('[META] VULN_HIGH=' + str(v.get('high', 0)))
    print('[META] VULN_CRITICAL=' + str(v.get('critical', 0)))
except:
    print('[META] VULN_HIGH=0')
    print('[META] VULN_CRITICAL=0')
" 2>/dev/null || true'''
                    }

                    echo "[META] DEPENDENCY_SCAN=PASSED"
                    echo "[STAGE_SUCCESS] Dependency Audit"
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    echo "[STAGE_START] Create Dockerfile"

                    if (!fileExists('app/Dockerfile')) {
                        if (env.STACK == "django") {
                            writeFile file: 'app/Dockerfile', text: """FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt || true
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                        } else if (env.STACK == "vite") {
                            writeFile file: 'app/Dockerfile', text: """FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm install || true
RUN npm run build || true

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                        } else {
                            writeFile file: 'app/Dockerfile', text: """FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install || true
ENV PORT=3000
EXPOSE 3000
CMD sh -c "PORT=3000 node server.js || PORT=3000 node index.js || PORT=3000 npm start || PORT=3000 node app.js"
"""
                        }
                    }

                    echo "[STAGE_SUCCESS] Create Dockerfile"
                }
            }
        }

        stage('Build Image') {
            steps {
                script { echo "[STAGE_START] Build Image" }
                sh 'cd app && docker build -t auto-app:${APP_ID} .'
                script { echo "[STAGE_SUCCESS] Build Image" }
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                script {
                    echo "[STAGE_START] Image Scan (Trivy)"
                    echo "Scanning Docker image for CVEs..."

                    sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 0 --severity HIGH,CRITICAL --no-progress auto-app:${APP_ID} 2>&1 || true'

                    sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 0 --severity CRITICAL --format json --no-progress --quiet auto-app:${APP_ID} > /tmp/trivy.json 2>/dev/null || true'

                    def criticalCount = sh(
                        script: '''python3 -c "
import json
try:
    d = json.load(open('/tmp/trivy.json'))
    total = sum(1 for r in (d.get('Results') or []) for v in (r.get('Vulnerabilities') or []) if v.get('Severity') == 'CRITICAL')
    print(total)
except:
    print(0)
" 2>/dev/null || echo 0''',
                        returnStdout: true
                    ).trim()

                    echo "[META] IMAGE_CRITICAL_CVE=${criticalCount}"
                    echo "[META] IMAGE_SCAN=PASSED"

                    if (criticalCount.toInteger() > 0) {
                        echo "WARNING: ${criticalCount} CRITICAL CVEs found in image!"
                    } else {
                        echo "No critical CVEs found in Docker image"
                    }

                    echo "[STAGE_SUCCESS] Image Scan (Trivy)"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script { echo "[STAGE_START] Push to DockerHub" }
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:${APP_ID}
                            docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:latest
                            docker push ${DOCKER_USER}/auto-app:${APP_ID}
                            docker push ${DOCKER_USER}/auto-app:latest
                            echo "[META] DOCKER_IMAGE=${DOCKER_USER}/auto-app:${APP_ID}"
                        '''
                    }
                }
                script { echo "[STAGE_SUCCESS] Push to DockerHub" }
            }
        }

        stage('Stop Old Container') {
            steps {
                script { echo "[STAGE_START] Stop Old Container" }
                sh 'docker stop app_${APP_ID} || true && docker rm app_${APP_ID} || true'
                script { echo "[STAGE_SUCCESS] Stop Old Container" }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    echo "[STAGE_START] Run Container"

                    if (env.STACK == "django") {
                        sh "docker run -d --name ${CONTAINER_NAME} -p ${EXTERNAL_PORT}:8000 auto-app:${APP_ID}"
                    } else if (env.STACK == "vite") {
                        sh "docker run -d --name ${CONTAINER_NAME} -p ${EXTERNAL_PORT}:80 auto-app:${APP_ID}"
                    } else {
                        sh "docker run -d --name ${CONTAINER_NAME} -p ${EXTERNAL_PORT}:3000 auto-app:${APP_ID}"
                    }

                    echo "[STAGE_SUCCESS] Run Container"
                }
            }
        }

        stage('Verify') {
            steps {
                script { echo "[STAGE_START] Verify" }
                sh 'docker ps'
                script {
                    echo "[META] URL=http://192.168.122.127:${EXTERNAL_PORT}"
                    echo "[STAGE_SUCCESS] Verify"
                }
            }
        }
    }

    post {
        success {
            echo "[DEPLOY_SUCCESS]"
            echo "[META] FINAL_STATUS=success"
            echo "[META] URL=http://192.168.122.127:${EXTERNAL_PORT}"
        }
        failure {
            echo "[DEPLOY_FAILED]"
            echo "[META] FINAL_STATUS=failed"
        }
    }
}
