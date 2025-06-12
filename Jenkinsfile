pipeline {
    agent any

    environment {
        IMAGE_NAME = "java-ci"
        DOCKER_REGISTRY = "anilk13"
        GIT_CREDENTIALS_ID = "git-cred"
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
                git branch: "${env.ACTUAL_BRANCH}",
                    url: 'https://github.com/anil78-ops/java-CI.git',
                    credentialsId: "${GIT_CREDENTIALS_ID}"
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
                withCredentials([usernamePassword(credentialsId: 'gitpushfor-updateyamlfile', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    script {
                        sh """
                            git config user.name "AIL6339"
                            git config user.email "vanilkumar4191@gmail.com"
                            git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/anil78-ops/java-CI.git
                            git add manifests/dev/dev-deployment.yaml
                            git commit -m "🔄 Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                            git push origin ${ACTUAL_BRANCH}
                        """
                    }
                }
            }
        }
    }
}
