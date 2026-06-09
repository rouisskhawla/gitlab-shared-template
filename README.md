# gitlab-shared-template

Reusable GitLab CI/CD job templates that provide shared pipeline definitions for **build**, **Docker**, and **Kubernetes deployment** workflows across multiple projects.

Projects include this repository's `pipeline.yml` and extend its job templates using GitLab CI's `extends` keyword. All logic lives here, each project only sets variables and job dependencies.

---

## Repository Structure

```
gitlab-shared-template/
└── pipeline.yml    # All reusable job templates
```

---

## Available Templates

| Template | Stage | Description |
|---|---|---|
| `.compute-version` | — | Computes the image version tag and writes it to `build.env` |
| `.build-backend-template` | `build` | Maven build for JDK 17 backend services |
| `.build-frontend-template` | `build` | Node.js 24 npm build for frontend services |
| `.docker-template` | `docker` | Docker-in-Docker image build and push |
| `.deploy-staging-template` | `deploy-staging` | Helm deploy to the `dev` namespace (runs on `dev` branch) |
| `.deploy-production-template` | `deploy-production` | Helm deploy to the `prod` namespace (manual gate, `main` branch only) |

---

## Usage

### Include the Template

In your service's `.gitlab-ci.yml`:

```yaml
include:
  - project: 'username/gitlab-shared-template'
    ref: main
    file: '/pipeline.yml'
```

### Extend the Templates

Backend service example:

```yaml
include:
  - project: 'username/gitlab-shared-template'
    ref: main
    file: '/pipeline.yml'

stages:
  - build
  - docker
  - deploy-staging
  - deploy-production

build-api-gateway:
  variables:
    SERVICE_DIR: services/api-gateway
  extends: .build-backend-template

docker-api-gateway:
  variables:
    SERVICE_DIR:  services/api-gateway
    DOCKER_IMAGE: username/ci-cd-gateway
  extends: .docker-template
  needs:
    - job: build-api-gateway
      artifacts: true

deploy-staging-api-gateway:
  variables:
    SERVICE_NAME: api-gateway
  extends: .deploy-staging-template
  needs:
    - job: docker-api-gateway
      artifacts: true

deploy-production-api-gateway:
  variables:
    SERVICE_NAME: api-gateway
  extends: .deploy-production-template
  needs:
    - job: docker-api-gateway
      artifacts: true
```

Frontend service extend `.build-frontend-template` instead:

```yaml
build-bookstore-frontend:
  variables:
    SERVICE_DIR: services/bookstore-frontend
  extends: .build-frontend-template
```

**Job name prefixes are required.** Because GitLab CI merges all included pipelines into a single root pipeline, job names must be unique across all services. Prefix every job with the service name (e.g. `build-api-gateway`, `build-books-service`).

---

## Template Variables

Each job passes variables to configure the template it extends:

| Variable | Used by | Description |
|---|---|---|
| `SERVICE_DIR` | `.build-backend-template`, `.build-frontend-template`, `.docker-template` | Path to the service directory |
| `DOCKER_IMAGE` | `.docker-template` | Docker image name |
| `SERVICE_NAME` | `.deploy-staging-template`, `.deploy-production-template` | Service name used in Helm release and namespace targeting |

---

## How VERSION Crosses Job Boundaries

GitLab CI jobs run in separate containers. `.compute-version` writes the computed tag to a `build.env` file and declares it as a `dotenv` artifact:

```yaml
artifacts:
  reports:
    dotenv: build.env
```

Downstream jobs (`.docker-template`, `.deploy-*-template`) consume it via:

```yaml
needs:
  - job: build-api-gateway
    artifacts: true
```

This makes `$VERSION` available as an environment variable in those jobs, the GitLab CI equivalent of GitHub Actions job outputs or Jenkins' `env.VERSION`.

---

## Version Format

| Branch | Example |
|---|---|
| `dev` | `1.0.47-dev-a3f9c12` |
| `main` | `1.0.47-a3f9c12` |

Format: `1.0.<pipeline_iid>[-dev]-<short_sha>`, traceable to a branch and commit without looking anything up.

---

## Docker-in-Docker

The `.docker-template` uses `docker:24.0.5-dind` as a service with TLS disabled. This is required because GitLab CI jobs run inside containers and do not have Docker available by default.

```yaml
services:
  - name: docker:24.0.5-dind
    command: ["--tls=false"]
variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""
```

Ensure your GitLab runner is configured to allow privileged mode for Docker-in-Docker to work.

---

## Manual Approval Gate

`.deploy-production-template` sets `when: manual`. On `main` branch pipelines, a play button appears in the GitLab pipeline UI that a human must click before the production Helm deploy runs.

---

## Root Pipeline Setup (Monorepo)

When using this template in a monorepo, scope each service's inclusion in the root `.gitlab-ci.yml` with `rules: changes:` so that only modified services trigger a pipeline run:

```yaml
# .gitlab-ci.yml (root)
include:
  - local: '/services/api-gateway/.gitlab-ci.yml'
    rules:
      - changes:
          - services/api-gateway/**/*
  - local: '/services/books-service/.gitlab-ci.yml'
    rules:
      - changes:
          - services/books-service/**/*
```

---

## Required CI/CD Variables

Configure these under **Settings → CI/CD → Variables** in your application repository:

| Variable | Notes |
|---|---|
| `DOCKER_USERNAME` | Plain text |
| `DOCKER_PASSWORD` | Masked |
| `KUBECONFIG_DEV` | Full kubeconfig content for dev cluster |
| `KUBECONFIG_PROD` | Full kubeconfig content for prod cluster |

---

## Related Repositories

- [reliable-ci-cd-pipeline](https://github.com/rouisskhawla/reliable-ci-cd-pipeline) — application monorepo that uses these templates
- [jenkins-shared-library](https://github.com/rouisskhawla/jenkins-shared-library) — equivalent pattern for Jenkins
- [github-shared-workflow](https://github.com/rouisskhawla/github-shared-workflow) — equivalent pattern for GitHub Actions
