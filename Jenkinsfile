pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'GitHub repository URL')
    }

    environment {
        DOCKERHUB_CRED = credentials('dockerhub-cred')
        DOCKERHUB_USER = 'raheeljamal'
    }

    stages {

        stage('Init') {
            steps {
                script {
                    echo '[STAGE_START] Init'
                    echo 'MULTI APP SAFE DEPLOY STARTED'
                    echo '[STAGE_SUCCESS] Init'
                }
            }
        }

        stage('Input Repo') {
            steps {
                script {
                    echo '[STAGE_START] Input Repo'

                    def repoUrl = params.REPO_URL?.trim() ?: 'https://github.com/RAHEEL-JAMAL/portfolio-website.git'

                    def appId = sh(
                        script: "printf '%s' '${repoUrl}' | md5sum | cut -c1-6",
                        returnStdout: true
                    ).trim()

                    env.REPO_URL       = repoUrl
                    env.APP_ID         = appId
                    env.CONTAINER_NAME = "app_${appId}"
                    env.IMAGE_NAME     = "${DOCKERHUB_USER}/${env.CONTAINER_NAME}:latest"

                    echo "[META] REPO_URL=${env.REPO_URL}"
                    echo "[META] APP_ID=${env.APP_ID}"
                    echo "[META] CONTAINER_NAME=${env.CONTAINER_NAME}"
                    echo "[META] IMAGE_NAME=${env.IMAGE_NAME}"
                    echo '[STAGE_SUCCESS] Input Repo'
                }
            }
        }

        stage('Allocate Safe Port') {
            steps {
                script {
                    echo '[STAGE_START] Allocate Safe Port'

                    def usedRaw = sh(
                        script: "docker ps --format '{{.Ports}}' | grep -o '[0-9]*->' | grep -o '[0-9]*' || true",
                        returnStdout: true
                    ).trim()

                    def usedPorts = usedRaw ? usedRaw.split('\n').toList() : []
                    def port = 3000

                    while (usedPorts.contains(port.toString())) {
                        port++
                    }

                    env.PORT = port.toString()

                    echo "[META] PORT=${env.PORT}"
                    echo '[STAGE_SUCCESS] Allocate Safe Port'
                }
            }
        }

  stage('Clone Repo') {
    steps {
        script {
            echo '[STAGE_START] Download Repo'
        }

        sh '''
            rm -rf app
            curl -L https://github.com/evertramos/docker-php-app/archive/refs/heads/main.zip -o app.zip
            unzip app.zip
            mv docker-php-app-main app
            rm app.zip
        '''

        script {
            echo '[STAGE_SUCCESS] Repo Downloaded'
        }
    }
}      
        
        
        stage('Setup Docker Ignore') {
            steps {
                script {
                    echo '[STAGE_START] Docker Ignore Setup'

                    sh '''
                        cat > app/.dockerignore <<EOF
node_modules
.git
npm-debug.log
dist
build
.env
coverage
.DS_Store
EOF
                    '''

                    echo '[META] DOCKERIGNORE_CREATED=true'
                    echo '[STAGE_SUCCESS] Docker Ignore Setup'
                }
            }
        }

        stage('Secret Scan') {
            steps {
                script {
                    echo '[STAGE_START] Secret Scan'
                    echo 'Scanning repo for hardcoded secrets...'
                }

                sh '''#!/bin/bash
                    FOUND=0

                    find app -type f \\( -name "*.js" -o -name "*.py" -o -name "*.env" \\) \
                        -not -path "*/node_modules/*" \
                        -not -path "*/.git/*" \
                        -not -name "package-lock.json" | while read -r f; do

                        if grep -qiE "(password|api_key|secret|access_token)\\s*=" "$f"; then
                            echo "[WARN] Possible secret in: $f"
                            FOUND=1
                        fi
                    done

                    if [ "$FOUND" = "1" ]; then
                        echo "[META] SECRET_SCAN=FAILED"
                        exit 1
                    fi

                    echo "No secrets found."
                    echo "[META] SECRET_SCAN=PASSED"
                '''

                script { echo '[STAGE_SUCCESS] Secret Scan' }
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    echo '[STAGE_START] Detect Stack'

                    def pkgRoot = 'app'
                    def stack   = 'unknown'

                    if (fileExists('app/package.json')) {
                        def pkg = readFile('app/package.json')

                        if      (pkg.contains('"vite"'))   stack = 'vite'
                        else if (pkg.contains('"next"'))   stack = 'nextjs'
                        else if (pkg.contains('"react"'))  stack = 'react'
                        else                               stack = 'node'
                    } else {

                        def mp = sh(script: "find app -maxdepth 2 -name manage.py | head -1", returnStdout: true).trim()
                        if (mp) {
                            stack = 'django'
                            pkgRoot = mp.replaceAll('/manage\\.py', '')
                        }

                        def cp = sh(script: "find app -maxdepth 2 -name composer.json | head -1", returnStdout: true).trim()
                        if (cp) {
                            stack = 'php'
                            pkgRoot = cp.replaceAll('/composer\\.json', '')
                        }

                        def jp = sh(script: "find app -maxdepth 2 \\( -name pom.xml -o -name build.gradle \\) | head -1", returnStdout: true).trim()
                        if (jp) {
                            stack = 'java'
                            pkgRoot = jp.replaceAll('/(pom\\.xml|build\\.gradle)', '')
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

        stage('Dependency Audit') {
            steps {
                script {
                    echo '[STAGE_START] Dependency Audit'
                }

                sh '''#!/bin/bash
                    set -e

                    if [ -f "${PKG_ROOT}/package.json" ]; then

                        ABS_PATH="$(cd "${PKG_ROOT}" && pwd)"

                        docker run --rm \
                            -v "${ABS_PATH}:/work" \
                            -w /work \
                            node:20-alpine \
                            sh -c "npm install --prefer-offline --no-fund 2>/dev/null; npm audit --json > /work/npm-audit.json 2>&1 || true"

                        echo "[META] DEPENDENCY_AUDIT=DONE"
                    else
                        echo "[META] DEPENDENCY_AUDIT=SKIPPED"
                    fi
                '''

                script { echo '[STAGE_SUCCESS] Dependency Audit' }
            }
        }

    stage('Create Dockerfile') {
    steps {
        script {
            echo '[STAGE_START] Create Dockerfile'

            // 🔥 Safety log (helps debugging in Jenkins)
            echo "[INFO] STACK=${env.STACK}"
            echo "[INFO] PKG_ROOT=${env.PKG_ROOT}"

            def df = ''

            switch (env.STACK) {

                case 'vite':
                case 'react':
                    df = '''FROM node:20-alpine AS builder
WORKDIR /app

# 🔥 FIX: install from correct context (monorepo safe)
COPY package*.json ./
RUN npm install --legacy-peer-deps

# copy full source after install
COPY . .

RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
'''
                    break

                case 'nextjs':
                    df = '''FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install --legacy-peer-deps

COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app

COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./

RUN npm install --omit=dev

EXPOSE 3000
CMD ["npm", "start"]
'''
                    break

                case 'node':
                    df = '''FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm install --legacy-peer-deps --omit=dev

COPY . .

EXPOSE 3000
CMD ["node", "index.js"]
'''
                    break

                case 'django':
                    df = '''FROM python:3.11-slim
WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
'''
                    break

                default:
                    df = '''FROM node:20-alpine
WORKDIR /app
COPY . .
EXPOSE 3000
CMD ["echo", "Unknown stack"]
'''
            }

            // 🔥 IMPORTANT: write into detected project root
            writeFile file: "${env.PKG_ROOT}/Dockerfile", text: df

            echo '[META] DOCKERFILE_CREATED=true'
            echo '[STAGE_SUCCESS] Create Dockerfile'
        }
    }
}
        stage('Detect App Root') {
    steps {
        script {
            echo '[STAGE_START] Detect App Root'

            sh '''
                echo "Scanning repo structure..."

                if [ -f package.json ]; then
                    echo "APP_ROOT=." > app.env
                elif [ -f app/package.json ]; then
                    echo "APP_ROOT=app" > app.env
                elif [ -f frontend/package.json ]; then
                    echo "APP_ROOT=frontend" > app.env
                else
                    echo "APP_ROOT=unknown" > app.env
                fi
            '''

            def props = readFile('app.env').trim()
            env.APP_ROOT = props.split('=')[1]

            echo "[META] APP_ROOT=${env.APP_ROOT}"
            echo '[STAGE_SUCCESS] Detect App Root'
        }
    }
}

    stage('Build Image') {
    steps {
        script { echo '[STAGE_START] Build Image' }

        sh """
            docker build --network=host \
            -t ${IMAGE_NAME} \
            ${APP_ROOT}
        """

        script { echo '[STAGE_SUCCESS] Build Image' }
    }
}

        stage('Image Scan (Trivy)') {
            steps {
                script { echo '[STAGE_START] Image Scan (Trivy)' }

                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --severity HIGH,CRITICAL \
                        "${IMAGE_NAME}"
                '''

                script { echo '[STAGE_SUCCESS] Image Scan (Trivy)' }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script { echo '[STAGE_START] Push to DockerHub' }

                retry(3) {
                    timeout(time: 15, unit: 'MINUTES') {
                        sh '''
                            echo "$DOCKERHUB_CRED_PSW" | docker login -u "$DOCKERHUB_CRED_USR" --password-stdin
                            docker push "${IMAGE_NAME}"
                            docker logout
                        '''
                    }
                }

                script { echo '[STAGE_SUCCESS] Push to DockerHub' }
            }
        }

        stage('Stop Old Container') {
            steps {
                script { echo '[STAGE_START] Stop Old Container' }
                sh '''
                    docker stop "${CONTAINER_NAME}" || true
                    docker rm "${CONTAINER_NAME}" || true
                '''
                script { echo '[STAGE_SUCCESS] Stop Old Container' }
            }
        }

        stage('Run Container') {
            steps {
                script { echo '[STAGE_START] Run Container' }
                sh 'docker run -d --name "${CONTAINER_NAME}" -p "${PORT}:80" "${IMAGE_NAME}"'
                script { echo '[STAGE_SUCCESS] Run Container' }
            }
        }

        stage('Verify') {
            steps {
                script { echo '[STAGE_START] Verify' }
                sh 'docker ps | grep "${CONTAINER_NAME}"'
                script { echo '[STAGE_SUCCESS] Verify' }
            }
        }
    }

    post {
        success { echo '[DEPLOY_SUCCESS]' }
        failure { echo '[DEPLOY_FAILED]' }
        always { echo '[META] Pipeline complete.' }
    }
}
