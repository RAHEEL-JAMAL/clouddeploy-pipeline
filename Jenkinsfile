stage('Detect Branch') {
    steps {
        script {

            // 1. If user provided branch → use it
            if (params.BRANCH?.trim()) {
                env.BRANCH_NAME = params.BRANCH.trim()
                echo "✔ Using user-provided branch: ${env.BRANCH_NAME}"
                return
            }

            // 2. Get raw output
            def raw = sh(
                script: "git ls-remote --heads ${params.REPO_URL}",
                returnStdout: true
            ).trim()

            echo "RAW OUTPUT:"
            echo raw

            // 3. Extract branches SAFELY
            def branchList = []

            raw.split("\n").each { line ->
                if (line.contains("refs/heads/")) {
                    def name = line.split("refs/heads/")[1].trim()
                    branchList.add(name)
                }
            }

            echo "Parsed branches: ${branchList}"

            // 4. FORCE LOGIC (no ambiguity)
            if (branchList.size() == 0) {
                error "❌ No branches found!"
            }

            if (branchList.contains("main")) {
                env.BRANCH_NAME = "main"
            } else if (branchList.contains("master")) {
                env.BRANCH_NAME = "master"
            } else {
                env.BRANCH_NAME = branchList[0]
            }

            echo "✅ FINAL SELECTED BRANCH: ${env.BRANCH_NAME}"
        }
    }
}
