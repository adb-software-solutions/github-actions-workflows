# ADB GitHub Actions Workflows

Reusable GitHub Actions workflows for ADB Software Solutions repositories.

The goal of this repository is to keep application repositories declarative: the application owns **when** a workflow runs and the small amount of configuration that genuinely differs, while this repository owns the repeated CI/CD implementation.

## Conventions

The workflows are designed around the conventions used by current ADB projects:

- Python 3.12.
- Node.js 22 and pnpm.
- Repository-local `tools/lint` as the source of truth for linting.
- Django backends under `backend/`.
- Frontends commonly under `website/` or `auth-frontend/`.
- Docker images in the DigitalOcean Container Registry.
- Trivy filesystem/image scanning.
- Infisical machine identities authenticated with GitHub OIDC.
- Production deployment through `adb-software-solutions/adb-deploy` and `deploy_application.yml`.

These are defaults, not hard requirements. Inputs exist for genuine differences between repositories.

## Reusable workflows

| Workflow | Purpose |
| --- | --- |
| `lint.yml` | Prepare the standard Python/Node/shell toolchain and run the repository's own `tools/lint`. |
| `django-ci.yml` | Run Django tests with PostgreSQL and Valkey, produce coverage, and optionally upload it. |
| `frontend-ci.yml` | Install pnpm dependencies, run frontend tests, and build the frontend. |
| `docker-build-push.yml` | Build one or more Docker images, authenticate to the ADB registry through Infisical, push, and scan them. |
| `deploy-application.yml` | Authenticate through OIDC, obtain deployment credentials/secrets from Infisical, and invoke `adb-deploy`. |
| `devcontainer-ci.yml` | Build a repository devcontainer as a CI smoke test. |

## Caller responsibility

Application repositories retain event and path filters. A normal caller should be deliberately small:

```yaml
name: Backend CI

on:
  push:
    branches: [main]
    paths:
      - "backend/**"
      - "Dockerfile.backend"
      - "tools/**"
  pull_request:
    paths:
      - "backend/**"
      - "Dockerfile.backend"
      - "tools/**"

jobs:
  lint:
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/lint.yml@v1

  backend:
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/django-ci.yml@v1
    with:
      django-settings-module: creatorclerk.settings
```

The shared workflows should never need individual secret values as inputs. Secret values stay in Infisical.

## OIDC and Infisical

Jobs that authenticate to Infisical require `id-token: write`. The GitHub OIDC token is issued in the context of the **calling repository**, even when the implementation is a reusable workflow. This preserves the per-application identity model.

The long-term trust policy can additionally bind `job_workflow_ref` so an application identity can only be used through approved workflows in this repository.

For pull requests from public repositories, secret-bearing jobs must not authenticate for fork PRs. The reusable workflows defensively avoid Infisical authentication for fork pull requests, but callers should also avoid creating secret-dependent jobs for untrusted fork code.

## Deployment secret mapping

`deploy-application.yml` deliberately avoids application-specific secret maps. Application secrets are read from the application's Infisical project and converted to Ansible extra-vars by convention:

```text
BACKEND_SECRET_KEY -> <application>_backend_secret_key
DB_PASSWORD        -> <application>_db_password
```

Shared platform secrets remain shared, for example `CLOUDFLARE_API_TOKEN -> cloudflare_api_token`.

This keeps application deploy callers small while allowing `vars/applications/<application>.yml` in `adb-deploy` to remain the deployment catalogue.

## Versioning

During development callers may reference a feature branch. Production callers should use a stable major tag such as `@v1` (or an immutable commit SHA where maximum reproducibility is required).

## Repository visibility

GitHub does not allow a **public** repository to call a reusable workflow stored in a **private** repository. Because TechWiki is public, this repository will need to be public before TechWiki can consume these workflows. The workflows must therefore contain no embedded credentials or secret values; all sensitive material is obtained at runtime through OIDC and Infisical.
