pipeline {
    agent any

    environment {
        APP_DIR = "app"
        SCANNER_HOME = tool 'sonar-scanner'
        // Convert branch name to safe Docker tag (e.g. release/v1 -> release-v1-42)
        IMAGE_TAG = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
        DOCKER_REPO = "anilk13/java-ci"
    }

    tools {
        jdk 'jdk17'
        maven 'Maven3'
    }

    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Branch Validation') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'uat'
                    branch pattern: "release/.*", comparator: "REGEXP"
                }
            }
            steps {
                echo "Running pipeline on allowed branch: ${BRANCH_NAME}"
            }
        }

        stage('Git Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'git-cred',
                        url: 'https://github.com/anil78-ops/java-CI.git'
                    ]]
                ])
            }
        }

        stage('Print Build Info') {
            steps {
                echo "Build Number: ${BUILD_NUMBER}"
                echo "Branch Name: ${BRANCH_NAME}"
                echo "Docker Image: ${DOCKER_REPO}:${IMAGE_TAG}"
                echo "Build URL: ${BUILD_URL}"
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('Maven Clean Install') {
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
                        sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=CI-JOB \
                        -Dsonar.projectKey=CI-JOB \
                        -Dsonar.java.binaries=target
                        '''
                    }
                }
            }
        }

        stage('Publish Artifacts') {
            steps {
                dir("${APP_DIR}") {
                    withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'Maven3', traceability: true) {
                        sh "mvn deploy"
                    }
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                dir("${APP_DIR}") {
                    script {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t ${DOCKER_REPO}:${IMAGE_TAG} ."
                        }
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push ${DOCKER_REPO}:${IMAGE_TAG}"
                    }
                }
            }
        }
    }
}
