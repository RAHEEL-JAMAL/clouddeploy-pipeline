Started by user Raheel Jamal
Lightweight checkout support not available, falling back to full checkout.
Checking out git https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git into /var/jenkins_home/workspace/fyp-pipeline@script/d1156cc66033b1a8275ad5d9d6049f7287596188ba3cb22fef223651ad78163e to read Jenkinsfile
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/fyp-pipeline@script/d1156cc66033b1a8275ad5d9d6049f7287596188ba3cb22fef223651ad78163e/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git # timeout=10
Fetching upstream changes from https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git
 > git --version # timeout=10
 > git --version # 'git version 2.47.3'
 > git fetch --tags --force --progress -- https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git +refs/heads/*:refs/remotes/origin/* # timeout=10
Seen branch in repository origin/main
Seen 1 remote branch
 > git show-ref --tags -d # timeout=10
Checking out Revision 4d952f9a01977ddda05c4b31564261e15a8fb6dd (origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 4d952f9a01977ddda05c4b31564261e15a8fb6dd # timeout=10
Commit message: "Add parameters and update Jenkins pipeline stages"
 > git rev-list --no-walk ff5071016dbf40a4bf767af9e80021b0b1216014 # timeout=10
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/fyp-pipeline
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/fyp-pipeline/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git # timeout=10
Fetching upstream changes from https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git
 > git --version # timeout=10
 > git --version # 'git version 2.47.3'
 > git fetch --tags --force --progress -- https://github.com/RAHEEL-JAMAL/clouddeploy-pipeline.git +refs/heads/*:refs/remotes/origin/* # timeout=10
Seen branch in repository origin/main
Seen 1 remote branch
 > git show-ref --tags -d # timeout=10
Checking out Revision 4d952f9a01977ddda05c4b31564261e15a8fb6dd (origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 4d952f9a01977ddda05c4b31564261e15a8fb6dd # timeout=10
Commit message: "Add parameters and update Jenkins pipeline stages"
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Init)
[Pipeline] echo
🚀 Deploying: myapp1
[Pipeline] echo
📦 Repo: https://github.com/RAHEEL-JAMAL/portfolio-website.git
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Clone Repo)
[Pipeline] sh
+ rm -rf app
+ git clone --depth 1 https://github.com/RAHEEL-JAMAL/portfolio-website.git app
Cloning into 'app'...
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Fix Frontend (SAFE NGINX ROUTING))
[Pipeline] sh
+ cd app
+ [ ! -f Dockerfile ]
+ cat
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Build Image (STABLE MODE))
[Pipeline] sh
+ cd app
+ DOCKER_BUILDKIT=0 docker build -t raheeljamal/myapp1:v1 .
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            BuildKit is currently disabled; enable it by removing the DOCKER_BUILDKIT=0
            environment-variable.

Sending build context to Docker daemon  8.959MB

Step 1/4 : FROM nginx:alpine
 ---> 812d47f806db
Step 2/4 : COPY dist /usr/share/nginx/html
COPY failed: file not found in build context or excluded by .dockerignore: stat dist: file does not exist
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Push Image (FIXED CREDENTIALS))
Stage "Push Image (FIXED CREDENTIALS)" skipped due to earlier failure(s)
[Pipeline] getContext
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Deploy Container (CLEAN + FIXED PORT))
Stage "Deploy Container (CLEAN + FIXED PORT)" skipped due to earlier failure(s)
[Pipeline] getContext
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Verify)
Stage "Verify" skipped due to earlier failure(s)
[Pipeline] getContext
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Declarative: Post Actions)
[Pipeline] echo
❌ DEPLOYMENT FAILED
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
ERROR: script returned exit code 1
Finished: FAILURE

