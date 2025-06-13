pipeline {
    agent any

    environment {
        IMAGE_NAME = "java-ci"
        DOCKER_REGISTRY = "anilk13"
        APP_DIR = "app"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        jdk 'jdk17'
        maven 'Maven3'
    }

    // Define a global variable to control skipping
    // This needs to be set in a way that it's accessible across stages for 'when' conditions.
    // The most reliable way for early checks is within a 'script' block at the top-level 'stages' or 'steps'
    // or by making the whole stages section conditional, which is not directly declarative.
    // A better approach is to use the `beforeAgent` which runs before an agent is allocated for stages.

    stages {
        stage('Determine Branch and Early Skip Check') {
            steps {
                script {
                    env.ACTUAL_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceFirst(/^origin\//, '') ?: 'dev'
                    echo "🔀 Detected branch: ${env.ACTUAL_BRANCH}"

                    // IMPORTANT: We need to determine the last committer *before* the main 'Clone Repository'
                    // The initial SCM checkout done by Jenkins (visible in your console output)
                    // has already provided the commit message and committer for the Jenkinsfile itself.
                    // We can try to leverage that.

                    // If your Jenkins job is triggered by a commit, the SCM plugin *usually*
                    // provides environmental variables like `GIT_COMMITTER_EMAIL` or similar.
                    // Let's try to get the last commit email from the initial SCM checkout info.
                    // If not, we'll need to do a quick fetch to get it.

                    def lastCommitEmail = ''
                    // Attempt to get commit email from initial SCM checkout (more efficient)
                    // This often requires accessing internal Jenkins variables or the `currentBuild` object
                    // to get the commit details. The `changelog` property of `currentBuild.changeSets`
                    // might contain this, but it's more complex to parse here.
                    // For simplicity, let's stick with git log, but ensure we have the context.

                    // Check if the current workspace has the git repository cloned from the initial checkout.
                    // The console output suggests it is: `/var/lib/jenkins/workspace/java-CI@script/...`
                    // We can execute `git log` from there.
                    try {
                        // Ensure we are in the correct directory for git commands.
                        // The initial checkout places it at the workspace root.
                        dir("${env.WORKSPACE}") { // Or just use `sh` without dir if already at root
                            sh "git fetch origin ${env.ACTUAL_BRANCH}" // Ensure branch is up-to-date
                            lastCommitEmail = sh(script: "git log -1 --pretty=format:'%ae'", returnStdout: true).trim()
                        }
                        echo "📧 Last commit by: ${lastCommitEmail}"

                        if (lastCommitEmail == 'vanilkumar4191@gmail.com') {
                            echo "🚫 Commit made by Jenkins bot, skipping entire build to prevent loop."
                            // Use `currentBuild.result` to mark the build status and `error` to stop the pipeline.
                            // The `post { success { ... } }` block will still run for this.
                            currentBuild.result = 'SUCCESS'
                            // This `error` will stop the pipeline immediately.
                            // The `post` section will then be executed.
                            error "Pipeline skipped due to Jenkins bot commit."
                        }
                    } catch (Exception e) {
                        echo "⚠️ Could not determine last commit email or error during early skip check: ${e.getMessage()}"
                        // If there's an error getting the commit info, proceed with the build.
                    }
                }
            }
        }

        stage('Branch Validation') {
            steps {
                script {
                    def allowed = env.ACTUAL_BRANCH == 'dev' ||
                                  env.ACTUAL_BRANCH == 'uat' ||
                                  env.ACTUAL_BRANCH.startsWith('release/') ||
                                  env.ACTUAL_BRANCH.startsWith('hotfix/')

                    if (!allowed) {
                        error "❌ Branch '${env.ACTUAL_BRANCH}' is not allowed."
                    }

                    echo "✅ Branch '${env.ACTUAL_BRANCH}' passed validation."
                }
            }
        }

        stage('Clone Repository') {
            steps {
                // This stage is still important even if we fetched earlier,
                // as it ensures the correct branch/commit is checked out for the rest of the build.
                git branch: "${env.ACTUAL_BRANCH}",
                    url: 'https://github.com/anil78-ops/java-CI.git',
                    credentialsId: 'gitpushfor-updateyamlfile' // Ensure this credential is valid for cloning
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('Maven Build') {
            steps {
                dir("${APP_DIR}") {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    dir("${APP_DIR}") {
                        sh """
                            ${SCANNER_HOME}/bin/sonar-scanner \\
                            -Dsonar.projectKey=CI-JOB \\
                            -Dsonar.projectName=CI-JOB \\
                            -Dsonar.java.binaries=target
                        """
                    }
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                dir("${APP_DIR}") {
                    script {
                        def safeTag = env.ACTUAL_BRANCH.replaceAll('/', '-')
                        def imageTag = "${safeTag}-${BUILD_NUMBER}"
                        env.IMAGE_TAG = imageTag

                        withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                            sh """
                                docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                                docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                script {
                    def manifestFile = 'manifests/dev/dev-deployment.yaml'

                    if (!fileExists(manifestFile)) {
                        error "❌ Manifest file not found at ${manifestFile}"
                    }

                    def imageLine = "image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    def updatedContent = readFile(file: manifestFile).replaceAll(/image: .*/, imageLine)
                    writeFile(file: manifestFile, text: updatedContent)

                    echo "✅ Updated deployment manifest with image tag: ${IMAGE_TAG}"
                }
            }
        }

        stage('Commit and Push Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'gitpushfor-updateyamlfile',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    script {
                        sh """
                            git config --global user.name "AIL6339"
                            git config --global user.email "vanilkumar4191@gmail.com"
                            git remote set-url origin https://${GIT_USER}:${GIT_PASSWORD}@github.com/anil78-ops/java-CI.git
                            git add manifests/dev/dev-deployment.yaml
                            git commit -m "🔄 Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                            git push origin ${ACTUAL_BRANCH}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Running post-build cleanup..."
            // Optional: Remove Trivy output
            sh "rm -f fs.html || true"

            // Optional: Clean Docker (adjust as needed, e.g., remove just built image)
            sh "docker image prune -f || true"

            // Optional: clean workspace
            cleanWs()
        }

        success {
            // This will execute if currentBuild.result was set to 'SUCCESS'
            // for the early exit, or if the entire pipeline ran successfully.
            echo "✅ Build completed successfully!"
        }

        failure {
            echo "❌ Build failed. Please check logs."
        }

        aborted {
            echo "🚫 Build was aborted."
        }
    }
}
