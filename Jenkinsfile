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

    stages {
        stage('Determine Branch') {
            steps {
                script {
                    env.ACTUAL_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceFirst(/^origin\//, '') ?: 'dev'
                    echo "🔀 Detected branch: ${env.ACTUAL_BRANCH}"
                }
            }
        }

        // Moved to be very early
        stage('Skip if Jenkins Commit (Early Check)') {
            steps {
                script {
                    // We need to clone just enough to check the last commit.
                    // Or, if you use a webhook, the payload might contain the committer info.
                    // For simplicity, let's clone the head.
                    // Note: This git clone might be redundant if the SCM already provides it.
                    // You might need to adjust this depending on your Jenkins SCM setup.
                    try {
                        sh "git fetch origin ${env.ACTUAL_BRANCH}"
                        def lastCommitEmail = sh(script: "git log -1 --pretty=format:'%ae'", returnStdout: true).trim()
                        echo "📧 Last commit by: ${lastCommitEmail}"
                        if (lastCommitEmail == 'vanilkumar4191@gmail.com') {
                            echo "🚫 Commit made by Jenkins bot, skipping build to prevent loop."
                            currentBuild.result = 'SUCCESS' // Mark current build as success
                            sh "exit 0" // This will exit the current shell step successfully
                            // You might need to use `error("Skipping build")` and configure global catch on error
                            // or use `return` if this block is inside a function that can return.
                            // A simpler way is to use `currentBuild.result = 'SUCCESS'` and then ensure no
                            // further stages run.
                            script {
                                // This is a common pattern to stop further stages
                                // if you want to completely abort the current pipeline run gracefully.
                                // `error` throws an exception, which will trigger the `failure` post-action.
                                // `currentBuild.result = 'SUCCESS'` and then ending execution here
                                // is tricky without `error` or a global return.
                                // The best way in a declarative pipeline to stop further stages and mark success
                                // is usually to use `error` and handle it in a `post` section.
                                // However, the intent here is to `return` from the script block,
                                // which only returns from the `script` step, not the entire pipeline.
                                // A more robust way to skip all subsequent stages is:
                                // If this is triggered by a Jenkins bot commit, we want to essentially make this build
                                // a "no-op" and mark it as successful.

                                // One way to achieve this is to make the entire pipeline conditionally run.
                                // But that's not possible directly in a declarative pipeline with top-level `when`.
                                // The existing `return` within the script block *does* exit that step.
                                // To prevent *subsequent stages* from running, you can use a global flag.
                                env.SKIP_FULL_BUILD = 'true'
                            }
                        }
                    } catch (Exception e) {
                        echo "⚠️ Could not determine last commit email or error during early skip check: ${e.getMessage()}"
                        // Continue with the build if there's an error in determining the last commit.
                        // This ensures builds still run if git log fails for some reason.
                    }
                }
            }
        }

        stage('Branch Validation') {
            when { expression { return env.SKIP_FULL_BUILD != 'true' } } // Only run if not skipping
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
            when { expression { return env.SKIP_FULL_BUILD != 'true' } } // Only run if not skipping
            steps {
                git branch: "${env.ACTUAL_BRANCH}",
                    url: 'https://github.com/anil78-ops/java-CI.git',
                    credentialsId: 'gitpushfor-updateyamlfile'
            }
        }

        // ... (Add `when { expression { return env.SKIP_FULL_BUILD != 'true' } }` to all subsequent stages)
        stage('Trivy FS Scan') {
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('Maven Build') {
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
            steps {
                dir("${APP_DIR}") {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
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
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
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
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
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
            when { expression { return env.SKIP_FULL_BUILD != 'true' } }
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
