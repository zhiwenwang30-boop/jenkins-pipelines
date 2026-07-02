pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  triggers {
    cron('* * * * *')
  }

  parameters {
    booleanParam(
      name: 'FORCE_RUN',
      defaultValue: false,
      description: 'Run even when the application repository commit did not change.'
    )
  }

  environment {
    APP_REPO_URL = 'https://github.com/zhiwenwang30-boop/helloworld.git'
    APP_BRANCH = 'main'
    APP_DIR = 'app'
    IMAGE_NAME = 'zhiwenwang30-boop/helloworld'
    GITHUB_CREDENTIALS_ID = 'github-push-cred'
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub-cred'
    PYTHON_MIRROR_IMAGE = 'docker.m.daocloud.io/library/python:3.12-slim'
  }

  stages {
    stage('Detect App Change') {
      steps {
        script {
          def latestCommit = sh(
            script: "git ls-remote '${env.APP_REPO_URL}' 'refs/heads/${env.APP_BRANCH}' | awk '{print \\$1}'",
            returnStdout: true
          ).trim()

          if (!latestCommit) {
            error("Could not read ${env.APP_BRANCH} from ${env.APP_REPO_URL}.")
          }

          env.TARGET_APP_COMMIT = latestCommit

          def checkpointFile = '.last-app-commit'
          def lastBuiltCommit = fileExists(checkpointFile) ? readFile(checkpointFile).trim() : ''
          def timerCauses = currentBuild.getBuildCauses('hudson.triggers.TimerTrigger$TimerTriggerCause')
          def isTimerBuild = timerCauses && timerCauses.size() > 0
          def forceRun = params.FORCE_RUN || !isTimerBuild

          if (!forceRun && !lastBuiltCommit) {
            env.SHOULD_RUN = 'false'
            writeFile file: checkpointFile, text: "${latestCommit}\n"
            currentBuild.displayName = "#${env.BUILD_NUMBER} baseline ${latestCommit.take(7)}"
            echo "Initialized polling baseline at ${latestCommit}; no pipeline run needed."
          } else if (!forceRun && lastBuiltCommit == latestCommit) {
            env.SHOULD_RUN = 'false'
            currentBuild.displayName = "#${env.BUILD_NUMBER} no app change"
            echo "No new app commit on ${env.APP_BRANCH}: ${latestCommit}"
          } else {
            env.SHOULD_RUN = 'true'
            echo "Application commit selected for build: ${latestCommit}"
          }
        }
      }
    }

    stage('Preflight Credentials') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: env.GITHUB_CREDENTIALS_ID,
            usernameVariable: 'GIT_USERNAME',
            passwordVariable: 'GIT_TOKEN'
          ),
          usernamePassword(
            credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
            usernameVariable: 'DOCKERHUB_USERNAME',
            passwordVariable: 'DOCKERHUB_TOKEN'
          )
        ]) {
          echo 'Required GitHub and Docker Hub credentials are configured.'
        }
      }
    }

    stage('Checkout App Repo') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          checkout([
            $class: 'GitSCM',
            branches: [[name: env.TARGET_APP_COMMIT]],
            userRemoteConfigs: [[
              url: env.APP_REPO_URL,
              credentialsId: env.GITHUB_CREDENTIALS_ID
            ]],
            extensions: [
              [$class: 'CleanBeforeCheckout']
            ]
          ])
        }
      }
    }

    stage('Prepare Version') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          script {
            def shortHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            def timestamp = sh(script: 'date +%Y%m%d-%H%M%S', returnStdout: true).trim()
            env.VERSION_TAG = "ci-${timestamp}-${shortHash}"
            env.IMAGE_VERSION = "${env.IMAGE_NAME}:${env.VERSION_TAG}"
            env.IMAGE_LATEST = "${env.IMAGE_NAME}:latest"
            currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.VERSION_TAG}"
          }
        }
      }
    }

    stage('Static Code Scan') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          sh '''
            tar \
              --exclude=.git \
              --exclude=.ruff_cache \
              --exclude=jenkins_home \
              -cf - . \
              | docker run --rm -i "${PYTHON_MIRROR_IMAGE}" \
                  sh -c "mkdir -p /workspace && tar -xf - -C /workspace && cd /workspace && pip install --no-cache-dir ruff bandit && ruff check main.py && bandit -q -r main.py"
          '''
        }
      }
    }

    stage('Build Docker Image') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          sh '''
            docker build -t "${IMAGE_VERSION}" .
            docker tag "${IMAGE_VERSION}" "${IMAGE_LATEST}"
          '''
        }
      }
    }

    stage('Run Smoke Test') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        sh '''
          output="$(docker run --rm "${IMAGE_VERSION}")"
          echo "$output"
          test "$output" = "Hello, World!"
        '''
      }
    }

    stage('Create Git Tag') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          sh '''
            git config user.name "jenkins-bot"
            git config user.email "jenkins@example.local"

            if git rev-parse "${VERSION_TAG}" >/dev/null 2>&1; then
              echo "Tag ${VERSION_TAG} already exists locally."
            else
              git tag "${VERSION_TAG}"
            fi
          '''
        }
      }
    }

    stage('Push Git Tag') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        dir(env.APP_DIR) {
          withCredentials([
            usernamePassword(
              credentialsId: env.GITHUB_CREDENTIALS_ID,
              usernameVariable: 'GIT_USERNAME',
              passwordVariable: 'GIT_TOKEN'
            )
          ]) {
            sh '''
              AUTH_REPO_URL=$(printf '%s' "${APP_REPO_URL}" | sed "s#https://#https://${GIT_USERNAME}:${GIT_TOKEN}@#")
              trap 'git remote set-url origin "${APP_REPO_URL}"' EXIT
              git remote set-url origin "${AUTH_REPO_URL}"
              git push origin "${VERSION_TAG}"
            '''
          }
        }
      }
    }

    stage('Push Docker Image') {
      when {
        expression { env.SHOULD_RUN == 'true' }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
            usernameVariable: 'DOCKERHUB_USERNAME',
            passwordVariable: 'DOCKERHUB_TOKEN'
          )
        ]) {
          sh '''
            printf '%s' "${DOCKERHUB_TOKEN}" | docker login -u "${DOCKERHUB_USERNAME}" --password-stdin
            docker push "${IMAGE_VERSION}"
            docker push "${IMAGE_LATEST}"
          '''
        }
      }
    }
  }

  post {
    success {
      script {
        if (env.SHOULD_RUN == 'true') {
          writeFile file: '.last-app-commit', text: "${env.TARGET_APP_COMMIT}\n"
          echo "Pipeline success: ${env.VERSION_TAG} was tagged and pushed."
        } else {
          echo 'Pipeline skipped because the application repository did not change.'
        }
      }
    }
    failure {
      echo 'Pipeline failed. Check the failed stage logs.'
    }
    always {
      sh 'docker logout || true'
    }
  }
}
