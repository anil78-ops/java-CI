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

    // stage('Publish Artifacts') {
    //   steps {
    //     dir("${APP_DIR}") {
    //       withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk17', maven: 'Maven3', traceability: true) {
    //         sh 'mvn deploy'
    //       }
    //     }
    //   }
    // }

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


stage('Kubernetes Deploy') {
  when {
    expression { return ['dev', 'uat'].contains(env.ACTUAL_BRANCH) || env.ACTUAL_BRANCH.startsWith('release/') || env.ACTUAL_BRANCH.startsWith('hotfix/') }
  }
  steps {
    script {
      def branch = env.ACTUAL_BRANCH
      def namespace = ['dev': 'dev', 'uat': 'uat'][branch] ?: 'prod'
      def deploymentFile = ""
      def kubeconfigCredentialId = ""
      def safeTag = branch.replaceAll('/', '-')
      def imageTag = "${safeTag}-${BUILD_NUMBER}"

      switch (branch) {
        case 'dev':
          kubeconfigCredentialId = 'kubeconfig-dev'
          deploymentFile = 'manifests/dev/dev-deployment.yaml'
          break
        case 'uat':
          kubeconfigCredentialId = 'kubeconfig-uat'
          deploymentFile = 'manifests/uat/uat-deployment.yaml'
          break
        default:
          kubeconfigCredentialId = 'kubeconfig-prod'
          deploymentFile = 'manifests/prod/prod-deployment.yaml'
      }

      withCredentials([
        file(credentialsId: kubeconfigCredentialId, variable: 'KCFG'),
        [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']
      ]) {
        sh '''
          export KUBECONFIG=$KCFG
          sed -i 's|image: .*|image: anilk13/java-ci:'"$IMAGE_TAG"'|' '''${deploymentFile}'''
          kubectl apply --insecure-skip-tls-verify -f '''${deploymentFile}'''
          kubectl rollout status --insecure-skip-tls-verify deployment/k8s-demo -n '''${namespace}'''
        '''
      }
    }
  }
}


  }

  post {
    always {
      echo "🧹 Running cleanup..."

      // Cleanup Trivy output
      sh 'rm -f fs.html || true'

      // Remove dangling images
      sh 'docker image prune -f || true'

      // Optional: remove this specific build image to save space
      script {
        def safeTag = env.ACTUAL_BRANCH.replaceAll('/', '-')
        def imageTag = "${safeTag}-${BUILD_NUMBER}"
        def fullImage = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${imageTag}"
        sh "docker rmi ${fullImage} || true"
      }

      // Cleanup workspace if needed
      cleanWs()
    }
  }
}
