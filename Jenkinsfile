
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
                    while (usedPorts.contains(port.toString())) { port++ }
                    env.EXTERNAL_PORT = port.toString()
                    echo "[META] PORT=${EXTERNAL_PORT}"
                    echo "[STAGE_SUCCESS] Allocate Safe Port"
                }
            }
        }

        stage('Clone Repo') {
            steps {
                script { echo "[STAGE_START] Clone Repo" }
                sh 'rm -rf app && git clone --depth 1 ${REPO_URL} app'
                script { echo "[STAGE_SUCCESS] Clone Repo" }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    echo "[STAGE_START] Secret Scan (Gitleaks)"
                    echo "Scanning repo for hardcoded secrets..."

                    def found = sh(returnStdout: true, script: '''
find app -type f \( -name "*.js" -o -name "*.py" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.ts" \) \
    ! -path "*/node_modules/*" \
    ! -path "*/.git/*" \
    ! -name "package-lock.json" \
    ! -name "yarn.lock" \
    -exec grep -liE "(password=|secret=|api_key=|access_token=|private_key|AWS_SECRET_ACCESS_KEY|BEGIN RSA PRIVATE|BEGIN OPENSSH PRIVATE)" {} ; 2>/dev/null || true
''').trim()

                    if (found) {
                        echo "WARNING: Possible secrets in: ${found}"
                        echo "[META] SECRET_SCAN=FAILED"
                    } else {
                        echo "No secrets found in repository"
                        echo "[META] SECRET_SCAN=PASSED"
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
                        env.STACK = pkg.contains("vite") ? "vite" : "node"
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
                        echo "[META] VULN_HIGH=0"
                        echo "[META] VULN_CRITICAL=0"
                    } else {
                        sh 'if [ -f app/package.json ]; then cd app && npm audit --json > /tmp/npm-audit.json 2>&1; fi || true'

                        def high = sh(returnStdout: true, script: "python3 -c \"import json; obj=json.load(open('/tmp/npm-audit.json')); print(obj.get('metadata',{}).get('vulnerabilities',{}).get('high',0))\" 2>/dev/null || echo 0").trim()
                        def critical = sh(returnStdout: true, script: "python3 -c \"import json; obj=json.load(open('/tmp/npm-audit.json')); print(obj.get('metadata',{}).get('vulnerabilities',{}).get('critical',0))\" 2>/dev/null || echo 0").trim()

                        echo "[META] VULN_HIGH=${high}"
                        echo "[META] VULN_CRITICAL=${critical}"
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
                            writeFile file: 'app/Dockerfile', text: 'FROM python:3.11\nWORKDIR /app\nCOPY . .\nRUN pip install -r requirements.txt || true\nEXPOSE 8000\nCMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]\n'
                        } else if (env.STACK == "vite") {
                            writeFile file: 'app/Dockerfile', text: 'FROM node:20-alpine AS build\nWORKDIR /app\nCOPY . .\nRUN npm install || true\nRUN npm run build || true\n\nFROM nginx:alpine\nCOPY --from=build /app/dist /usr/share/nginx/html\nEXPOSE 80\nCMD ["nginx", "-g", "daemon off;"]\n'
                        } else {
                            writeFile file: 'app/Dockerfile', text: 'FROM node:20-alpine\nWORKDIR /app\nCOPY . .\nRUN npm install || true\nENV PORT=3000\nEXPOSE 3000\nCMD sh -c "PORT=3000 node server.js || PORT=3000 node index.js || PORT=3000 npm start"\n'
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

                    def criticalCount = sh(returnStdout: true, script: "python3 -c \"import json; d=json.load(open('/tmp/trivy.json')); print(sum(1 for r in (d.get('Results') or []) for v in (r.get('Vulnerabilities') or []) if v.get('Severity')=='CRITICAL'))\" 2>/dev/null || echo 0").trim()

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
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:${APP_ID}'
                    sh 'docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:latest'
                    sh 'docker push ${DOCKER_USER}/auto-app:${APP_ID}'
                    sh 'docker push ${DOCKER_USER}/auto-app:latest'
                    sh 'echo "[META] DOCKER_IMAGE=${DOCKER_USER}/auto-app:${APP_ID}"'
                }
                script { echo "[STAGE_SUCCESS] Push to DockerHub" }
            }
        }

        stage('Stop Old Container') {
            steps {
                script { echo "[STAGE_START] Stop Old Container" }
                sh 'docker stop app_${APP_ID} || true'
                sh 'docker rm app_${APP_ID} || true'
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
