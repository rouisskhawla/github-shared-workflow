# github-shared-workflow

Reusable GitHub Actions workflows that standardize CI/CD automation across repositories using shared **build**, **Docker**, and **Kubernetes deployment** pipelines.

Each project calls the shared workflow with a few inputs. All build, packaging, and deployment logic lives here. In one place, versioned and maintained independently of the applications that use it.

---

## Repository Structure

```
github-shared-workflow/
└── .github/
    └── workflows/
        └── ci-cd-pipeline.yml    # The reusable workflow
```

---

## Usage

### Caller Workflow (in application repo)

Backend service:

```yaml
name: CI CD Pipeline Api Gateway

on:
  push:
    branches: [main, dev]
    paths:
      - 'services/api-gateway/**'

jobs:
  pipeline:
    uses: username/github-shared-workflow/.github/workflows/ci-cd-pipeline.yml@main
    with:
      service-name: api-gateway
      docker-image: username/ci-cd-gateway
      service-dir:  services/api-gateway
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      KUBECONFIG_DEV:  ${{ secrets.KUBECONFIG_DEV }}
      KUBECONFIG_PROD: ${{ secrets.KUBECONFIG_PROD }}
```

Frontend service (add `service-type: frontend`):

```yaml
jobs:
  pipeline:
    uses: username/github-shared-workflow/.github/workflows/ci-cd-pipeline.yml@main
    with:
      service-name: bookstore-frontend
      docker-image: username/ci-cd-frontend
      service-dir:  services/bookstore-frontend
      service-type: frontend
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      KUBECONFIG_DEV:  ${{ secrets.KUBECONFIG_DEV }}
      KUBECONFIG_PROD: ${{ secrets.KUBECONFIG_PROD }}
```

Use `paths:` to scope each caller workflow to its own service directory. A push that only touches `services/books-service/` will not trigger any other service's pipeline.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service-name` | ✅ | — | Service name used in Helm release names |
| `docker-image` | ✅ | — | Docker image name |
| `service-dir` | ✅ | — | Path to the service within the repository |
| `service-type` | ❌ | `backend` | `backend` (Maven/JDK) or `frontend` (Node.js/npm) |
| `java-version` | ❌ | `17` | JDK version for backend builds |

## Required Secrets

Secrets must be defined in the **calling repository**, not in this one:

| Secret | Description |
|---|---|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub password or access token |
| `KUBECONFIG_DEV` | kubeconfig file content for the dev cluster |
| `KUBECONFIG_PROD` | kubeconfig file content for the prod cluster |

---

## Pipeline Jobs

### `build`

Runs version computation, application build, Docker build, and Docker push.

**Version format:**

| Branch | Example Tag |
|---|---|
| `dev` | `1.0.47-dev-a3f9c12` |
| `main` | `1.0.47-a3f9c12` |

**Backend build:** sets up JDK with `actions/setup-java@v4`, runs `mvn clean package -DskipTests`.

**Frontend build:** sets up Node.js 24 with `actions/setup-node@v4`, runs `npm ci` then `npm run build --configuration development` (dev) or `--configuration production` (main).

**Docker:** builds the image, pushes the versioned tag, and on `main` also tags and pushes `latest`.

The computed image tag is published as a job output (`image-tag`) and consumed by the `deploy` job.

### `deploy`

Runs after `build` completes (`needs: build`). Reads the image tag from the `build` job output.

- Selects the kubeconfig, Helm values file, and Kubernetes namespace based on the branch (`dev` → `dev`, `main` → `prod`)
- Runs `helm upgrade --install` against the target cluster
- Confirms the rollout with `kubectl rollout status --timeout=5m`

**Manual approval gate:** the `deploy` job is assigned to a GitHub environment (`production` on `main`, `development` on `dev`). Requires human approval before deployments proceed.

---

## How Job Outputs Cross Job Boundaries

The `build` job publishes the version tag:

```yaml
outputs:
  image-tag: ${{ steps.version.outputs.tag }}
```

The `deploy` job consumes it:

```yaml
needs: build
# ...
--set global.imageTag=${{ needs.build.outputs.image-tag }}
```

The two jobs run on separate runner instances, declared outputs are the only way to pass data between them.

---

## Runner and Secrets Location

- The **self-hosted runner** is registered in the **calling repository** (the application monorepo), not in this repository.
- All **secrets** (`DOCKER_USERNAME`, etc.) are defined in the **calling repository**.
- This repository holds no secrets and requires no runner registration.

---

## Related Repositories

- [reliable-ci-cd-pipeline](https://github.com/rouisskhawla/reliable-ci-cd-pipeline) — application monorepo that calls this workflow
- [jenkins-shared-library](https://github.com/rouisskhawla/jenkins-shared-library) — equivalent pattern for Jenkins
- [gitlab-shared-template](https://github.com/rouisskhawla/gitlab-shared-template) — equivalent pattern for GitLab CI
