# Jenkins Shared Libraries

A collection of reusable **Jenkins shared library** steps (Groovy) for building CI/CD pipelines. Instead of copying the same stages into every `Jenkinsfile`, load this library once and call each stage as a single step.

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins&logoColor=white)
![Groovy](https://img.shields.io/badge/Groovy-4298B8?logo=apachegroovy&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?logo=aquasecurity&logoColor=white)

---

## Available Steps

| Step | Description | Parameters |
|------|-------------|------------|
| `clone` | Clones a Git repository and branch | `url`, `branch` |
| `run_tests` | Runs unit tests (placeholder, add your own test command) | none |
| `trivy_scan` | Scans the filesystem for vulnerabilities with Trivy (`trivy fs .`) | none |
| `docker_build` | Builds a Docker image and tags it with a version and `latest` | `imageName`*, `imageTag`, `dockerfile`, `context` |
| `docker_push` | Logs in to Docker Hub and pushes the version tag and `latest` | `imageName`*, `imageTag`, `credentials` |
| `update_k8s_manifests` | Updates image tags in Kubernetes manifests, then commits and pushes | `imageTag`*, `manifestsPath`, `gitCredentials`, `gitUserName`, `gitUserEmail` |
| `generate_reports` | Writes a build report to `reports/` and archives it | `projectName`, `imageName`, `imageTag` |
| `clean_ws` | Cleans the Jenkins workspace | none |

\* required

### Defaults

| Parameter | Default |
|-----------|---------|
| `imageTag` | `latest` |
| `dockerfile` | `Dockerfile` |
| `context` | `.` |
| `credentials` (Docker Hub) | `docker-hub-credentials` |
| `manifestsPath` | `kubernetes` |
| `gitCredentials` | `github-credentials` |
| `gitUserName` | `Jenkins CI` |

## Repository Structure

Jenkins loads global steps from the `vars/` directory, so keep the Groovy files there:

```
Jenkins-shared-libs/
└── vars/
    ├── clean_ws.groovy
    ├── clone.groovy
    ├── docker_build.groovy
    ├── docker_push.groovy
    ├── generate_reports.groovy
    ├── run_tests.groovy
    ├── trivy_scan.groovy
    └── update_k8s_manifests.groovy
```

## Setup

### 1. Register the library in Jenkins

1. Go to **Manage Jenkins → System → Global Trusted Pipeline Libraries**.
2. Click **Add** and set:
   - **Name:** `Shared` (or any name you prefer)
   - **Default version:** `main`
   - **Retrieval method:** Modern SCM → Git
   - **Project repository:** `https://github.com/imVikash-ai/Jenkins-shared-libs.git`
3. Save.

### 2. Create the required credentials

| Credential ID | Type | Used by |
|---------------|------|---------|
| `docker-hub-credentials` | Username with password | `docker_push` |
| `github-credentials` | Username with password (use a PAT) | `update_k8s_manifests` |

### 3. Install tools on the Jenkins agent

- Docker
- [Trivy](https://aquasecurity.github.io/trivy/)
- Git

## Usage

Load the library at the top of your `Jenkinsfile` and call the steps:

```groovy
@Library('Shared') _

pipeline {
    agent any

    environment {
        IMAGE_NAME = 'your-dockerhub-user/your-app'
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {
        stage('Clone') {
            steps { clone('https://github.com/your-user/your-app.git', 'main') }
        }

        stage('Unit Tests') {
            steps { run_tests() }
        }

        stage('Security Scan') {
            steps { trivy_scan() }
        }

        stage('Build Image') {
            steps {
                docker_build(
                    imageName: env.IMAGE_NAME,
                    imageTag:  env.IMAGE_TAG
                )
            }
        }

        stage('Push Image') {
            steps {
                docker_push(
                    imageName: env.IMAGE_NAME,
                    imageTag:  env.IMAGE_TAG
                )
            }
        }

        stage('Update Manifests') {
            steps { update_k8s_manifests(imageTag: env.IMAGE_TAG) }
        }
    }

    post {
        always {
            generate_reports(
                projectName: 'My App',
                imageName:   env.IMAGE_NAME,
                imageTag:    env.IMAGE_TAG
            )
            clean_ws()
        }
    }
}
```

## Customization Notes

- **`update_k8s_manifests`** currently contains hardcoded values (image names, manifest file names, ingress host, and the Git remote URL). Edit these to match your own project before using it.
- **`run_tests`** is a placeholder. Add your own command, such as `npm test` or `mvn test`.
- **`docker_build`** and **`docker_push`** always tag and push `latest` in addition to the version tag.

## Typical CI/CD Flow

```
Clone → Test → Trivy Scan → Docker Build → Docker Push → Update K8s Manifests → Report → Clean Workspace
```

Pairs well with a GitOps tool such as Argo CD, which picks up the updated manifests and deploys them.

## Contributing

Suggestions and improvements are welcome. Fork the repo, create a branch, and open a pull request.

## Author

**Vikash** — [@imVikash-ai](https://github.com/imVikash-ai)