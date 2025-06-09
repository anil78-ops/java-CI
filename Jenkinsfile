pipeline {
    agent any

    environment {
        APP_DIR = "app"
        SCANNER_HOME = tool 'sonar-scanner'
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
                echo "Running pipeline on allowed branch: ${env.BRANCH_NAME}"
            }
        }

        stage('Git Checkout') {
            steps {
                // This uses Jenkins' built-in checkout for the current branch
                checkout scm
            }
        }

        stage('Print Build Info') {
            steps {
                script {
                    def imageTag = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
                    echo "Build Number : ${env.BUILD_NUMBER}"
                    echo "Branch Name  : ${env.BRANCH_NAME}"
                    echo "Docker Image : ${DOCKER_REPO}:${imageTag}"
                    echo "Build URL    : ${env.BUILD_URL}"
                }
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
                        def imageTag = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t ${DOCKER_REPO}:${imageTag} ."
                        }
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    def imageTag = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push ${DOCKER_REPO}:${imageTag}"
                    }
                }
            }
        }
    }
}
