pipeline {
    agent any

    environment {
        IMAGE_NAME = "java-ci"
        DOCKER_REGISTRY = "anilk13"
        GIT_CREDENTIALS_ID = "git-cred"
        APP_DIR = "app"
        SCANNER_HOME = tool 'sonar-scanner' // Ensure 'sonar-scanner' is configured in Jenkins Global Tool Configuration
    }

    tools {
        jdk 'jdk17' // Ensure 'jdk17' is configured in Jenkins Global Tool Configuration
        maven 'Maven3' // Ensure 'Maven3' is configured in Jenkins Global Tool Configuration
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
                        error "❌ Branch '${env.ACTUAL_BRANCH}' is not allowed. Only dev, uat, release/*, and hotfix/* are permitted."
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
                // Ensure Trivy is installed and accessible in the Jenkins agent's PATH
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
                withSonarQubeEnv('sonar-server') { // Ensure 'sonar-server' is configured in Jenkins System Configuration
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

        // stage('Publish Artifacts') {
        //    steps {
        //        dir("${APP_DIR}") {
        //            withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'Maven3', traceability: true) {
        //                sh 'mvn deploy'
        //            }
        //        }
        //    }
        // }

        stage('Docker Build and Push') {
            steps {
                dir("${APP_DIR}") {
                    script {
                        def safeTag = env.ACTUAL_BRANCH.replaceAll('/', '-')
                        def imageTag = "${safeTag}-${BUILD_NUMBER}"
                        env.IMAGE_TAG = imageTag

                        // Ensure 'docker-cred' is a Docker Hub or other Docker Registry credential in Jenkins
                        // The 'url' parameter for withDockerRegistry is often not needed for Docker Hub or can be an empty string.
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
    }
}

stage('Update Deployment Manifest') {
    steps {
        script {
            def manifestFile = 'manifests/dev/dev-deployment.yaml'
            def imageLine = "image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
            def updatedContent = readFile(file: manifestFile).replaceAll(/image: .*/, imageLine)
            writeFile(file: manifestFile, text: updatedContent)

            echo "✅ Updated deployment manifest with image tag: ${IMAGE_TAG}"
        }
    }
}


