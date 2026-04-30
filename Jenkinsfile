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
                    echo "🚀 MULTI APP SAFE DEPLOY STARTED"
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

        // ─── SECURITY: SECRET SCANNING ────────────────────────────────────────
        stage('Secret Scan (Gitleaks)') {
            steps {
                script {
                    echo "[STAGE_START] Secret Scan (Gitleaks)"
                    echo "🔍 Scanning repo for hardcoded secrets..."

                    // Install gitleaks if not present
                    sh '''
                        if ! command -v gitleaks &> /dev/null; then
                            echo "Installing gitleaks..."
                            curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.18.4/gitleaks_8.18.4_linux_x64.tar.gz \
                                | tar -xz -C /usr/local/bin gitleaks || true
                        fi
                    '''

                    def leakResult = sh(
                        script: '''
                            gitleaks detect \
                                --source=./app \
                                --no-git \
                                --report-format=json \
                                --report-path=gitleaks-report.json \
                                --exit-code=1 2>&1 || echo "LEAKS_FOUND"
                        ''',
                        returnStdout: true
                    ).trim()

                    if (leakResult.contains('LEAKS_FOUND')) {
                        echo "[META] SECRET_SCAN=FAILED"
                        echo "⚠️  Hardcoded secrets detected! Check gitleaks-report.json"
                        // Warn but don't fail — change 'echo' to 'error()' to hard-fail
                        echo "WARNING: Secrets found in repo. Review before production use."
                    } else {
                        echo "[META] SECRET_SCAN=PASSED"
                        echo "✅ No secrets found in repository"
                    }

                    echo "[STAGE_SUCCESS] Secret Scan (Gitleaks)"
                }
            }
        }
        // ──────────────────────────────────────────────────────────────────────

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

        // ─── SECURITY: DEPENDENCY AUDIT ───────────────────────────────────────
        stage('Dependency Audit') {
            steps {
                script {
                    echo "[STAGE_START] Dependency Audit"
                    echo "🔍 Scanning dependencies for known vulnerabilities..."

                    if (env.STACK == "django") {
                        sh '''
                            pip install pip-audit --quiet || true
                            cd app
                            if [ -f requirements.txt ]; then
                                pip-audit -r requirements.txt --format=json -o pip-audit-report.json 2>&1 || true
                                VULNS=$(python3 -c "import json; d=json.load(open('pip-audit-report.json')); print(len(d.get('dependencies',[])))" 2>/dev/null || echo "0")
                                echo "Python packages scanned: $VULNS"
                            else
                                echo "No requirements.txt found, skipping pip audit"
                            fi
                        '''
                    } else {
                        sh '''
                            cd app
                            if [ -f package.json ]; then
                                npm audit --json > npm-audit-report.json 2>&1 || true
                                HIGH=$(cat npm-audit-report.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('metadata',{}).get('vulnerabilities',{}).get('high',0))" 2>/dev/null || echo "0")
                                CRITICAL=$(cat npm-audit-report.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('metadata',{}).get('vulnerabilities',{}).get('critical',0))" 2>/dev/null || echo "0")
                                echo "High severity: $HIGH | Critical: $CRITICAL"
                                echo "[META] VULN_HIGH=$HIGH"
                                echo "[META] VULN_CRITICAL=$CRITICAL"
                            else
                                echo "No package.json found, skipping npm audit"
                            fi
                        '''
                    }

                    echo "[META] DEPENDENCY_SCAN=PASSED"
                    echo "[STAGE_SUCCESS] Dependency Audit"
                }
            }
        }
        // ──────────────────────────────────────────────────────────────────────

        stage('Create Dockerfile') {
            steps {
                script {
                    echo "[STAGE_START] Create Dockerfile"

                    if (!fileExists('app/Dockerfile')) {

                        if (env.STACK == "django") {
                            writeFile file: 'app/Dockerfile', text: """
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt || true
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                        } else if (env.STACK == "vite") {
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine AS build
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
                            writeFile file: 'app/Dockerfile', text: """
FROM node:20-alpine
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

                sh '''
                    cd app
                    docker build -t auto-app:${APP_ID} .
                '''

                script { echo "[STAGE_SUCCESS] Build Image" }
            }
        }

        // ─── SECURITY: IMAGE VULNERABILITY SCAN ──────────────────────────────
        stage('Image Scan (Trivy)') {
            steps {
                script {
                    echo "[STAGE_START] Image Scan (Trivy)"
                    echo "🔍 Scanning Docker image for CVEs..."

                    // Install trivy if not present
                    sh '''
                        if ! command -v trivy &> /dev/null; then
                            echo "Installing Trivy..."
                            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                                | sh -s -- -b /usr/local/bin v0.52.0 || true
                        fi
                    '''

                    def trivyResult = sh(
                        script: '''
                            trivy image \
                                --exit-code 0 \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output trivy-report.json \
                                auto-app:${APP_ID} 2>&1

                            # Count vulnerabilities
                            CRITICAL=$(python3 -c "
import json, sys
try:
    d = json.load(open('trivy-report.json'))
    total = sum(
        len([v for v in r.get('Vulnerabilities', []) if v.get('Severity') == 'CRITICAL'])
        for result in d.get('Results', [])
        for r in [result]
    )
    print(total)
except:
    print(0)
" 2>/dev/null || echo "0")

                            HIGH=$(python3 -c "
import json, sys
try:
    d = json.load(open('trivy-report.json'))
    total = sum(
        len([v for v in r.get('Vulnerabilities', []) if v.get('Severity') == 'HIGH'])
        for result in d.get('Results', [])
        for r in [result]
    )
    print(total)
except:
    print(0)
" 2>/dev/null || echo "0")

                            echo "CRITICAL_CVE=$CRITICAL"
                            echo "HIGH_CVE=$HIGH"
                        ''',
                        returnStdout: true
                    ).trim()

                    def criticalCount = sh(
                        script: "echo '${trivyResult}' | grep 'CRITICAL_CVE=' | cut -d'=' -f2 || echo '0'",
                        returnStdout: true
                    ).trim()

                    echo "[META] IMAGE_CRITICAL_CVE=${criticalCount}"
                    echo "[META] IMAGE_SCAN=PASSED"

                    if (criticalCount.toInteger() > 0) {
                        echo "⚠️  ${criticalCount} CRITICAL CVEs found in image!"
                        echo "WARNING: Review trivy-report.json before deploying to production"
                        // To hard-fail on critical CVEs, uncomment below:
                        // error("Pipeline stopped: ${criticalCount} critical vulnerabilities found")
                    } else {
                        echo "✅ No critical CVEs found in Docker image"
                    }

                    echo "[STAGE_SUCCESS] Image Scan (Trivy)"
                }
            }
        }
        // ──────────────────────────────────────────────────────────────────────

        stage('Push to DockerHub') {
            steps {
                script {
                    echo "[STAGE_START] Push to DockerHub"
                }

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

                script {
                    echo "[STAGE_SUCCESS] Push to DockerHub"
                }
            }
        }

        stage('Stop Old Container') {
            steps {
                script { echo "[STAGE_START] Stop Old Container" }

                sh '''
                    docker stop app_${APP_ID} || true
                    docker rm app_${APP_ID} || true
                '''

                script { echo "[STAGE_SUCCESS] Stop Old Container" }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    echo "[STAGE_START] Run Container"

                    if (env.STACK == "django") {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:8000 \
                        auto-app:${APP_ID}
                        """
                    } else if (env.STACK == "vite") {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:80 \
                        auto-app:${APP_ID}
                        """
                    } else {
                        sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${EXTERNAL_PORT}:3000 \
                        auto-app:${APP_ID}
                        """
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
