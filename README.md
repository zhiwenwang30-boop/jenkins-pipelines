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

Trigger mode:

- Jenkins polls `https://github.com/zhiwenwang30-boop/helloworld.git`
  every minute with `* * * * *`.
- If `refs/heads/main` did not change since the last successful build, the
  pipeline skips all build/push stages.
- If `refs/heads/main` changed, the pipeline checks out that exact commit,
  creates a Git tag, builds the Docker image, and pushes both.

No public webhook URL is required for the local Jenkins demo environment.
