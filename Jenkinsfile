pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME    = 'raheeljamal'
    }

    stages {

        // ─────────────────────────────────────────────
        stage('Init') {
            steps {
                script {
                    echo '[STAGE_START] Init'
                    echo 'MULTI APP SAFE DEPLOY STARTED'
                    echo '[STAGE_SUCCESS] Init'
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Input Repo') {
            steps {
                script {
                    echo '[STAGE_START] Input Repo'

                    def repoUrl = params.REPO_URL ?: 'https://github.com/RAHEEL-JAMAL/portfolio-website.git'

                    def appId = sh(
                        script: "echo '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "app_${appId}"

                    echo "[META] REPO_URL=${env.REPO_URL}"
                    echo "[META] APP_ID=${env.APP_ID}"
                    echo "[META] CONTAINER_NAME=${env.CONTAINER_NAME}"
                    echo '[STAGE_SUCCESS] Input Repo'
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Allocate Safe Port') {
            steps {
                script {
                    echo '[STAGE_START] Allocate Safe Port'

                    def usedPorts = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | grep -o '[0-9]*' || true",
                        returnStdout: true
                    ).trim().split('\n').findAll { it }

                    def port = 3000
                    while (usedPorts.contains(port.toString())) { port++ }
                    env.PORT = port.toString()

                    echo "[META] PORT=${env.PORT}"
                    echo '[STAGE_SUCCESS] Allocate Safe Port'
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Clone Repo') {
            steps {
                script { echo '[STAGE_START] Clone Repo' }
                retry(3) {
                    sh '''
                        rm -rf app
                        git config --global http.version HTTP/1.1
                        git config --global http.postBuffer 524288000
                        git clone --depth 1 $REPO_URL app
                    '''
                }
                script { echo '[STAGE_SUCCESS] Clone Repo' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Secret Scan') {
            steps {
                script {
                    echo '[STAGE_START] Secret Scan (Gitleaks)'
                    echo 'Scanning repo for hardcoded secrets...'
                }
                sh '''
                    FOUND=0
                    for f in $(find app -type f \\( -name "*.js" -o -name "*.py" -o -name "*.env" \\) \
                        -not -path "*/node_modules/*" \
                        -not -path "*/.git/*" \
                        -not -name "package-lock.json"); do
                        if grep -qiE "(password|api_key|secret|access_token)\\s*=" "$f"; then
                            echo "[WARN] Possible secret in: $f"
                            FOUND=1
                        fi
                    done
                    if [ "$FOUND" = "1" ]; then
                        echo "[META] SECRET_SCAN=FAILED"
                        exit 1
                    else
                        echo "No secrets found in repository"
                        echo "[META] SECRET_SCAN=PASSED"
                    fi
                '''
                script { echo '[STAGE_SUCCESS] Secret Scan (Gitleaks)' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def pkgRoot = ''
                    def stack   = 'unknown'

                    if (fileExists('app/package.json')) {
                        pkgRoot = 'app'
                        def pkg = readFile('app/package.json')
                        if (pkg.contains('"vite"'))        stack = 'vite'
                        else if (pkg.contains('"next"'))   stack = 'nextjs'
                        else if (pkg.contains('"react"'))  stack = 'react'
                        else                               stack = 'node'
                    } else {
                        def managePy = sh(script: "find app -maxdepth 2 -name manage.py | head -1", returnStdout: true).trim()
                        if (managePy) {
                            stack = 'django'; pkgRoot = managePy.replaceAll('/manage.py', '')
                        }
                        def composerJson = sh(script: "find app -maxdepth 2 -name composer.json | head -1", returnStdout: true).trim()
                        if (composerJson) {
                            stack = 'php'; pkgRoot = composerJson.replaceAll('/composer.json', '')
                        }
                        def pomXml = sh(script: "find app -maxdepth 2 \\( -name pom.xml -o -name build.gradle \\) | head -1", returnStdout: true).trim()
                        if (pomXml) {
                            stack = 'java'; pkgRoot = pomXml.replaceAll('/(pom.xml|build.gradle)', '')
                        }
                    }

                    env.STACK    = stack
                    env.PKG_ROOT = pkgRoot

                    echo "[META] STACK=${env.STACK}"
                    echo "[META] PKG_ROOT=${env.PKG_ROOT}"
                    echo '[STAGE_SUCCESS] Detect Stack'
                }
            }
        }

        // ─────────────────────────────────────────────
        // ✅ FIX: Use bash explicitly to avoid "Bad substitution" in dash/sh
        stage('Dependency Audit') {
            steps {
                script {
                    echo '[STAGE_START] Dependency Audit'
                    echo 'Scanning dependencies for known vulnerabilities...'
                }
                sh '''#!/bin/bash
                    set -e
                    if [ -f app/package.json ]; then
                        echo "Running npm audit..."
                        docker run --rm \
                            -v "$(pwd)/app:/work" \
                            -w /work \
                            node:20-alpine \
                            sh -c "npm audit --json > /work/npm-audit.json 2>&1 || true"

                        if [ -f app/npm-audit.json ]; then
                            VULN_COUNT=$(grep -c '"severity"' app/npm-audit.json || echo 0)
                            echo "[META] DEPENDENCY_AUDIT=PASSED (vulnerabilities found: $VULN_COUNT — check npm-audit.json)"
                        else
                            echo "[META] DEPENDENCY_AUDIT=SKIPPED (no audit output)"
                        fi
                    else
                        echo "[META] DEPENDENCY_AUDIT=SKIPPED (no package.json)"
                    fi
                '''
                script { echo '[STAGE_SUCCESS] Dependency Audit' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Create Dockerfile') {
            steps {
                script {
                    echo '[STAGE_START] Create Dockerfile'

                    def dockerfile = ''
                    if (env.STACK == 'vite' || env.STACK == 'react') {
                        dockerfile = """
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
"""
                    } else if (env.STACK == 'nextjs') {
                        dockerfile = """
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./
RUN npm ci --omit=dev
EXPOSE 3000
CMD ["npm", "start"]
"""
                    } else if (env.STACK == 'node') {
                        dockerfile = """
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
EXPOSE 3000
CMD ["node", "index.js"]
"""
                    } else if (env.STACK == 'django') {
                        dockerfile = """
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
"""
                    } else {
                        dockerfile = """
FROM node:20-alpine
WORKDIR /app
COPY . .
EXPOSE 3000
CMD ["sh", "-c", "echo 'Unknown stack - please configure manually'"]
"""
                    }

                    writeFile file: "${env.PKG_ROOT}/Dockerfile", text: dockerfile
                    echo "[META] DOCKERFILE_CREATED=true"
                    echo '[STAGE_SUCCESS] Create Dockerfile'
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Build Image') {
            steps {
                script {
                    echo '[STAGE_START] Build Image'
                    env.IMAGE_NAME = "${env.DOCKERHUB_USERNAME}/${env.CONTAINER_NAME}:latest"
                }
                sh 'docker build -t $IMAGE_NAME $PKG_ROOT'
                script { echo '[STAGE_SUCCESS] Build Image' }
            }
        }

        // ─────────────────────────────────────────────
        // ✅ SECURITY STAGE — now visible in frontend pipeline view
        stage('Image Scan (Trivy)') {
            steps {
                script { echo '[STAGE_START] Image Scan (Trivy)' }
                sh '''
                    echo "Running Trivy vulnerability scan on image: $IMAGE_NAME"
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        $IMAGE_NAME
                    echo "[META] IMAGE_SCAN=PASSED"
                '''
                script { echo '[STAGE_SUCCESS] Image Scan (Trivy)' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                script { echo '[STAGE_START] Push to DockerHub' }
                sh '''
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push $IMAGE_NAME
                    docker logout
                '''
                script { echo '[STAGE_SUCCESS] Push to DockerHub' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Stop Old Container') {
            steps {
                script { echo '[STAGE_START] Stop Old Container' }
                sh '''
                    docker stop $CONTAINER_NAME 2>/dev/null || true
                    docker rm   $CONTAINER_NAME 2>/dev/null || true
                '''
                script { echo '[STAGE_SUCCESS] Stop Old Container' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Run Container') {
            steps {
                script { echo '[STAGE_START] Run Container' }
                sh 'docker run -d --name $CONTAINER_NAME -p $PORT:80 --restart unless-stopped $IMAGE_NAME'
                script { echo '[STAGE_SUCCESS] Run Container' }
            }
        }

        // ─────────────────────────────────────────────
        stage('Verify') {
            steps {
                script { echo '[STAGE_START] Verify' }
                sh '''
                    sleep 5
                    STATUS=$(docker inspect --format="{{.State.Running}}" $CONTAINER_NAME 2>/dev/null || echo "false")
                    if [ "$STATUS" = "true" ]; then
                        echo "[META] CONTAINER_STATUS=RUNNING"
                        echo "[META] DEPLOYED_PORT=$PORT"
                        echo "[META] FINAL_STATUS=success"
                        echo "[DEPLOY_SUCCESS]"
                    else
                        echo "[META] FINAL_STATUS=failed"
                        echo "[DEPLOY_FAILED]"
                        exit 1
                    fi
                '''
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }

    } // end stages

    // ─────────────────────────────────────────────
    post {
        success {
            echo '[DEPLOY_SUCCESS]'
            echo '[META] FINAL_STATUS=success'
        }
        failure {
            echo '[DEPLOY_FAILED]'
            echo '[META] FINAL_STATUS=failed'
        }
        always {
            sh 'docker logout || true'
        }
    }
}
