# Jenkins Pipelines

This repository stores Jenkins pipeline definitions outside application
repositories.

## Helloworld pipeline

Jenkins job configuration:

- Type: Pipeline
- Definition: Pipeline script from SCM
- SCM URL: `https://github.com/zhiwenwang30-boop/jenkins-pipelines.git`
- Branch: `main`
- Script Path: `pipelines/helloworld.Jenkinsfile`

Required Jenkins credentials:

- `github-push-cred`: GitHub username and personal access token.
- `dockerhub-cred`: Docker Hub username and access token.

Webhook on the application repository:

- Repository: `https://github.com/zhiwenwang30-boop/helloworld.git`
- URL: `http://localhost:18080/generic-webhook-trigger/invoke?token=helloworld-ci-token`
- Event: push

The pipeline only processes pushes to `refs/heads/main`. Tag pushes are ignored
to avoid creating a trigger loop.
