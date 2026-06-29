pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  triggers {
    GenericTrigger(
      genericVariables: [
        [key: 'WEBHOOK_REF', value: '$.ref'],
        [key: 'WEBHOOK_AFTER', value: '$.after']
      ],
      causeString: 'Triggered by GitHub push on $WEBHOOK_REF at $WEBHOOK_AFTER',
      token: 'helloworld-ci-token',
      printContributedVariables: true,
      printPostContent: false,
      regexpFilterText: '$WEBHOOK_REF',
      regexpFilterExpression: '^refs/heads/main$'
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
    stage('Validate Trigger') {
      steps {
        script {
          if (env.WEBHOOK_REF?.trim() && env.WEBHOOK_REF != 'refs/heads/main') {
            currentBuild.result = 'NOT_BUILT'
            error("Ignored ref: ${env.WEBHOOK_REF}")
          }
        }
      }
    }

    stage('Checkout App Repo') {
      steps {
        dir(env.APP_DIR) {
          checkout([
            $class: 'GitSCM',
            branches: [[name: env.WEBHOOK_AFTER?.trim() ? env.WEBHOOK_AFTER : "*/${env.APP_BRANCH}"]],
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
      steps {
        sh '''
          output="$(docker run --rm "${IMAGE_VERSION}")"
          echo "$output"
          test "$output" = "Hello, World!"
        '''
      }
    }

    stage('Create Git Tag') {
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
      echo "Pipeline success: ${VERSION_TAG} was tagged and pushed."
    }
    failure {
      echo 'Pipeline failed. Check the failed stage logs.'
    }
    always {
      sh 'docker logout || true'
    }
  }
}
