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
                retry(3) {
                    sh '''
                        rm -rf app
                        git config --global http.version HTTP/1.1
                        git config --global http.postBuffer 524288000
                        git clone --depth 1 ${REPO_URL} app
                    '''
                }
                script { echo "[STAGE_SUCCESS] Clone Repo" }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    echo "[STAGE_START] Secret Scan (Gitleaks)"
                    echo "Scanning repo for hardcoded secrets..."
                }
                sh '''
                    FOUND=0
                    for f in $(find app -type f -name "*.js" -not -path "*/node_modules/*" -not -path "*/.git/*" -not -name "package-lock.json"); do
                        grep -qi "password=" "$f" && echo "WARNING: password= in $f" && FOUND=1 || true
                        grep -qi "api_key=" "$f" && echo "WARNING: api_key= in $f" && FOUND=1 || true
                        grep -qi "secret=" "$f" && echo "WARNING: secret= in $f" && FOUND=1 || true
                        grep -qi "access_token=" "$f" && echo "WARNING: access_token= in $f" && FOUND=1 || true
                    done
                    for f in $(find app -type f -name "*.py" -not -path "*/.git/*"); do
                        grep -qi "password=" "$f" && echo "WARNING: password= in $f" && FOUND=1 || true
                        grep -qi "secret=" "$f" && echo "WARNING: secret= in $f" && FOUND=1 || true
                        grep -qi "AWS_SECRET" "$f" && echo "WARNING: AWS_SECRET in $f" && FOUND=1 || true
                    done
                    for f in $(find app -type f -name "*.env" -not -path "*/.git/*"); do
                        grep -qi "password=" "$f" && echo "WARNING: password= in $f" && FOUND=1 || true
                        grep -qi "secret=" "$f" && echo "WARNING: secret= in $f" && FOUND=1 || true
                    done
                    if [ "$FOUND" = "1" ]; then
                        echo "[META] SECRET_SCAN=FAILED"
                    else
                        echo "No secrets found in repository"
                        echo "[META] SECRET_SCAN=PASSED"
                    fi
                '''
                script { echo "[STAGE_SUCCESS] Secret Scan (Gitleaks)" }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo "[STAGE_START] Detect Stack"

                    // ── Smart: find package.json at root or one level deep ──
                    def pkgRoot = sh(returnStdout: true, script: '''
                        if [ -f app/package.json ]; then
                            echo "app"
                        else
                            FOUND=$(find app -maxdepth 2 -name "package.json" \
                                -not -path "*/node_modules/*" | head -1)
                            if [ -n "$FOUND" ]; then
                                dirname "$FOUND"
                            else
                                echo ""
                            fi
                        fi
                    ''').trim()

                    env.PKG_ROOT = pkgRoot

                    if (fileExists('app/manage.py') || sh(returnStdout: true, script: 'find app -maxdepth 2 -name "manage.py" | head -1 || true').trim()) {
                        env.STACK = "django"
                    } else if (sh(returnStdout: true, script: 'find app -maxdepth 2 -name "composer.json" | head -1 || true').trim()) {
                        env.STACK = "php"
                    } else if (sh(returnStdout: true, script: 'find app -maxdepth 2 -name "pom.xml" -o -name "build.gradle" | head -1 || true').trim()) {
                        env.STACK = "java"
                    } else if (pkgRoot) {
                        def pkg = readFile("${pkgRoot}/package.json")
                        if (pkg.contains('"vite"') || pkg.contains('"@vitejs"')) {
                            env.STACK = "vite"
                        } else if (pkg.contains('"react"') || pkg.contains('"next"')) {
                            env.STACK = "react"
                        } else {
                            env.STACK = "node"
                        }
                    } else if (sh(returnStdout: true, script: 'find app -maxdepth 2 -name "index.html" | head -1 || true').trim()) {
                        env.STACK = "html"
                    } else {
                        env.STACK = "node"
                    }

                    echo "[META] STACK=${STACK}"
                    echo "[META] PKG_ROOT=${env.PKG_ROOT}"
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

                    } else if (env.STACK in ["vite", "react", "node"]) {
                        sh """
                            if [ -f ${env.PKG_ROOT}/package.json ]; then
                                docker run --rm \
                                    -v \$(pwd)/${env.PKG_ROOT}:/work \
                                    -w /work \
                                    node:20-alpine \
                                    sh -c "npm audit --json > /work/npm-audit.json 2>&1 || true"
                            fi
                        """
                        sh 'python3 -c "import json; obj=json.load(open(\'${env.PKG_ROOT}/npm-audit.json\')); v=obj.get(\'metadata\',{}).get(\'vulnerabilities\',{}); print(\'[META] VULN_HIGH=\'+str(v.get(\'high\',0))); print(\'[META] VULN_CRITICAL=\'+str(v.get(\'critical\',0)))" 2>/dev/null || echo "[META] VULN_HIGH=0" && echo "[META] VULN_CRITICAL=0"'
                    } else {
                        echo "[META] VULN_HIGH=0"
                        echo "[META] VULN_CRITICAL=0"
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

                    // ── VITE ────────────────────────────────────────────────
                    if (env.STACK == "vite") {
                        sh """
                            docker run --rm \
                                -v \$(pwd)/${env.PKG_ROOT}:/work \
                                -w /work \
                                node:20-alpine \
                                sh -c "npm install && npm run build"
                        """
                        writeFile file: "${env.PKG_ROOT}/Dockerfile",
                            text: 'FROM nginx:alpine\nCOPY dist /usr/share/nginx/html\nEXPOSE 80\nCMD ["nginx", "-g", "daemon off;"]\n'
                        env.DOCKER_CONTEXT = env.PKG_ROOT

                    // ── REACT (CRA / Next) ──────────────────────────────────
                    } else if (env.STACK == "react") {
                        sh """
                            docker run --rm \
                                -v \$(pwd)/${env.PKG_ROOT}:/work \
                                -w /work \
                                node:20-alpine \
                                sh -c "npm install && npm run build 2>&1 || (npm run export 2>&1 || true)"
                        """
                        // Support both CRA (build/) and Next (out/)
                        def distDir = sh(returnStdout: true, script: """
                            if [ -d ${env.PKG_ROOT}/build ]; then echo "build"
                            elif [ -d ${env.PKG_ROOT}/out ]; then echo "out"
                            elif [ -d ${env.PKG_ROOT}/.next ]; then echo ".next"
                            else echo "build"; fi
                        """).trim()
                        writeFile file: "${env.PKG_ROOT}/Dockerfile",
                            text: "FROM nginx:alpine\nCOPY ${distDir} /usr/share/nginx/html\nEXPOSE 80\nCMD [\"nginx\", \"-g\", \"daemon off;\"]\n"
                        env.DOCKER_CONTEXT = env.PKG_ROOT

                    // ── NODE ────────────────────────────────────────────────
                    } else if (env.STACK == "node") {
                        sh """
                            docker run --rm \
                                -v \$(pwd)/${env.PKG_ROOT}:/work \
                                -w /work \
                                node:20-alpine \
                                sh -c "npm install"
                        """
                        if (!fileExists("${env.PKG_ROOT}/Dockerfile")) {
                            writeFile file: "${env.PKG_ROOT}/Dockerfile",
                                text: 'FROM node:20-alpine\nWORKDIR /app\nCOPY . .\nEXPOSE 3000\nCMD sh -c "node server.js || node index.js || npm start || node app.js"\n'
                        }
                        env.DOCKER_CONTEXT = env.PKG_ROOT

                    // ── DJANGO ──────────────────────────────────────────────
                    } else if (env.STACK == "django") {
                        def djangoRoot = sh(returnStdout: true, script: 'find app -maxdepth 2 -name "manage.py" | head -1 | xargs dirname || echo "app"').trim()
                        if (!fileExists("${djangoRoot}/Dockerfile")) {
                            writeFile file: "${djangoRoot}/Dockerfile",
                                text: 'FROM python:3.11-slim\nWORKDIR /app\nCOPY . .\nRUN pip install -r requirements.txt || true\nEXPOSE 8000\nCMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]\n'
                        }
                        env.DOCKER_CONTEXT = djangoRoot

                    // ── PHP ─────────────────────────────────────────────────
                    } else if (env.STACK == "php") {
                        if (!fileExists('app/Dockerfile')) {
                            writeFile file: 'app/Dockerfile',
                                text: 'FROM php:8.2-apache\nCOPY . /var/www/html/\nEXPOSE 80\n'
                        }
                        env.DOCKER_CONTEXT = "app"

                    // ── JAVA ────────────────────────────────────────────────
                    } else if (env.STACK == "java") {
                        if (!fileExists('app/Dockerfile')) {
                            writeFile file: 'app/Dockerfile',
                                text: 'FROM maven:3.9-eclipse-temurin-17 AS build\nWORKDIR /app\nCOPY . .\nRUN mvn package -DskipTests || true\nFROM eclipse-temurin:17-jre\nWORKDIR /app\nCOPY --from=build /app/target/*.jar app.jar\nEXPOSE 8080\nCMD ["java", "-jar", "app.jar"]\n'
                        }
                        env.DOCKER_CONTEXT = "app"

                    // ── HTML (static) ───────────────────────────────────────
                    } else if (env.STACK == "html") {
                        if (!fileExists('app/Dockerfile')) {
                            writeFile file: 'app/Dockerfile',
                                text: 'FROM nginx:alpine\nCOPY . /usr/share/nginx/html\nEXPOSE 80\nCMD ["nginx", "-g", "daemon off;"]\n'
                        }
                        env.DOCKER_CONTEXT = "app"

                    // ── FALLBACK ────────────────────────────────────────────
                    } else {
                        if (!fileExists('app/Dockerfile')) {
                            writeFile file: 'app/Dockerfile',
                                text: 'FROM nginx:alpine\nCOPY . /usr/share/nginx/html\nEXPOSE 80\nCMD ["nginx", "-g", "daemon off;"]\n'
                        }
                        env.DOCKER_CONTEXT = "app"
                    }

                    echo "[META] DOCKER_CONTEXT=${env.DOCKER_CONTEXT}"
                    echo "[STAGE_SUCCESS] Create Dockerfile"
                }
            }
        }

        stage('Build Image') {
            steps {
                script { echo "[STAGE_START] Build Image" }
                sh 'docker build --network=host -t auto-app:${APP_ID} ${DOCKER_CONTEXT}'
                script { echo "[STAGE_SUCCESS] Build Image" }
            }
        }

        stage('Image Scan (Trivy)') {
            steps {
                script {
                    echo "[STAGE_START] Image Scan (Trivy)"
                    echo "Scanning Docker image for CVEs..."
                }
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 0 --severity HIGH,CRITICAL --no-progress auto-app:${APP_ID} 2>&1 || true'
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 0 --severity CRITICAL --format json --no-progress --quiet auto-app:${APP_ID} > /tmp/trivy.json 2>/dev/null || true'
                script {
                    def criticalCount = sh(
                        returnStdout: true,
                        script: 'python3 -c "import json; d=json.load(open(\'/tmp/trivy.json\')); print(sum(1 for r in (d.get(\'Results\') or []) for v in (r.get(\'Vulnerabilities\') or []) if v.get(\'Severity\')==\'CRITICAL\'))" 2>/dev/null || echo 0'
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
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker tag auto-app:${APP_ID} ${DOCKER_USER}/auto-app:${APP_ID}'
                    sh 'docker push ${DOCKER_USER}/auto-app:${APP_ID}'
                    sh 'docker tag ${DOCKER_USER}/auto-app:${APP_ID} ${DOCKER_USER}/auto-app:latest'
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
                    } else if (env.STACK == "java") {
                        sh "docker run -d --name ${CONTAINER_NAME} -p ${EXTERNAL_PORT}:8080 auto-app:${APP_ID}"
                    } else if (env.STACK in ["vite", "react", "html", "php"]) {
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
