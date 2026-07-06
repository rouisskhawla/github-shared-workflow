# github-shared-workflow

Reusable GitHub Actions CI/CD workflows used to standardize build, test, security scanning, Docker image creation, and Kubernetes deployment across multiple repositories.

This repository acts as a **central CI/CD pipeline engine** consumed by application repositories via GitHub Actions reusable workflows.

---

## Architecture

```text
Application Repo
   └── calls reusable workflow
          ↓
github-shared-workflow
   ├── validate (tests, Sonar, dependency checks)
   ├── build (Maven / Node)
   ├── docker build & push
   ├── security scan (Trivy)
   ├── deploy (Helm to Kubernetes)
   └── notifications (Slack)
```

---

## Repository Structure

```text
.github/workflows/
├── ci-cd-pipeline.yml   # Main reusable CI/CD pipeline
├── app-validate.yml     # Testing + quality + dependency scanning
└── terraform-validate.yml (infra validation)
```

---

## How It Works

This repo is **not a standalone pipeline**.
It is executed by other repositories using:

```yaml
jobs:
  pipeline:
    uses: username/github-shared-workflow/.github/workflows/ci-cd-pipeline.yml@main
```

Each application defines:

* service name
* source directory
* Docker image
* build type (backend/frontend)

---

## Pipeline Flow

```text
1. Test & Quality
   - Unit tests
   - SonarQube analysis
   - Dependency scanning (OWASP / npm audit)

2. Build
   - Maven / Node build
   - Versioning

3. Container
   - Docker build & push

4. Security
   - Trivy image scan

5. Deploy
   - Helm deployment to Kubernetes
   - Dev / Prod separation

6. Notify
   - Slack notification (success/failure)
```

---

## Inputs

| Input        | Required | Description                  |
| ------------ | -------- | ---------------------------- |
| service-name | Yes      | Service / Helm release name  |
| service-dir  | Yes      | Source code directory        |
| service-key  | Yes      | SonarQube project key        |
| docker-image | Yes      | Docker image repository      |
| service-type | No       | backend (default) / frontend |

---

## Secrets Required

* Docker credentials
* Kubernetes kubeconfig (dev/prod)
* SonarQube token + URL
* NVD API key (OWASP scans)
* Slack bot token + channel ID

---

## Deployment Strategy

| Branch | Environment | Namespace |
| ------ | ----------- | --------- |
| dev    | Development | dev       |
| main   | Production  | prod      |

Production deployments require GitHub Environment approval.

---

## Key Features

* Reusable CI/CD across multiple repositories
* Standardized build and deployment process
* Security scanning (dependency + container)
* Kubernetes-based deployments using Helm
* Slack notifications for visibility
* Separation of dev and production environments

---

## Related Repositories

* [https://github.com/rouisskhawla/jenkins-shared-library](https://github.com/rouisskhawla/jenkins-shared-library) - Reusable Jenkins CI/CD library
* [https://github.com/rouisskhawla/gitlab-shared-template](https://github.com/rouisskhawla/gitlab-shared-template) - Reusable GitLab CI/CD templates
* [https://github.com/rouisskhawla/reliable-ci-cd-pipeline](https://github.com/rouisskhawla/reliable-ci-cd-pipeline) - Code repository
* [https://github.com/rouisskhawla/devops-infrastructure-terraform](https://github.com/rouisskhawla/devops-infrastructure-terraform) - Kubernetes infrastructure with Terraform and CI/CD integration
