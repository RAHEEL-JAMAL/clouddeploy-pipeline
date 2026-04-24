pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'Git repository URL')
        string(name: 'APP_NAME', defaultValue: 'myapp', description: 'App name')
        string(name: 'BRANCH', defaultValue: '', description: 'Branch (optional)')
        string(name: 'DEPLOYMENT_ID', defaultValue: 'v1', description: 'Tag')
    }

    environment {
        DOCKER_CREDS = credentials('dockerhub-cred')
    }

    stages {

        stage('Init') {
            steps {
                script {
                    env.SAFE_APP = params.APP_NAME.replaceAll(/[^a-zA-Z0-9]/, "").toLowerCase()
                    if (!env.SAFE_APP) {
                        env.SAFE_APP = "myapp"
                    }

                    echo "✔ APP: ${env.SAFE_APP}"
                }
            }
        }

        stage('Detect Branch') {
            steps {
                script {

                    def raw = sh(
                        script: "git ls-remote --heads ${params.REPO_URL}",
                        returnStdout: true
                    ).trim()

                    def branches = []

                    raw.split("\n").each { line ->
                        if (line.contains("refs/heads/")) {
                            branches.add(line.split("refs/heads/")[1].trim())
                        }
                    }

                    echo "Branches: ${branches}"

                    if (params.BRANCH?.trim()) {
                        env.BRANCH_NAME = params.BRANCH
                    } else if (branches.contains("main")) {
                        env.BRANCH_NAME = "main"
                    } else if (branches.contains("master")) {
                        env.BRANCH_NAME = "master"
                    } else {
                        env.BRANCH_NAME = branches[0]
                    }

                    echo "✔ Branch selected: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone') {
            steps {
                sh """
                    rm -rf /tmp/${env.SAFE_APP}
                    git clone --depth 1 -b ${env.BRANCH_NAME} ${params.REPO_URL} /tmp/${env.SAFE_APP}
                """
            }
        }

        stage('Detect Stack') {
            steps {
                script {
                    if (fileExists("/tmp/${env.SAFE_APP}/package.json")) {
                        env.STACK = "node"
                    } else if (fileExists("/tmp/${env.SAFE_APP}/requirements.txt")) {
                        env.STACK = "python"
                    } else {
                        env.STACK = "node"
                    }

                    echo "✔ Stack: ${env.STACK}"
                }
            }
        }
    }

    post {
        success {
            echo "🚀 SUCCESS"
        }
        failure {
            echo "❌ FAILED"
        }
    }
}
