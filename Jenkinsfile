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
                        env.ACTUAL_BRANCH ==~ /^release\/.*/ ||
                        env.ACTUAL_BRANCH ==~ /^hotfix\/.*/

          if (!allowed) {
            error "❌ Branch '${env.ACTUAL_BRANCH}' is not allowed. Allowed: dev, uat, release/*, hotfix/*"
          } else {
            echo "✅ Branch '${env.ACTUAL_BRANCH}' is allowed to proceed."
          }
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
              ${SCANNER_HOME}/bin/sonar-scanner \
              -Dsonar.projectKey=CI-JOB \
              -Dsonar.projectName=CI-JOB \
              -Dsonar.java.binaries=target
            """
          }
        }
      }
    }

    stage('Publish Artifacts') {
      steps {
        dir("${APP_DIR}") {
          withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'Maven3', traceability: true) {
            sh 'mvn deploy'
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
  }
}
