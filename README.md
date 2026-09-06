# ADB GitHub Actions Workflows

Reusable GitHub Actions workflows for ADB Software Solutions repositories.

This repository is intentionally public. It contains workflow implementation only; credentials and application secret values remain in Infisical and are obtained at runtime through GitHub OIDC.

The goal is to keep application repositories declarative: the application owns **when** a workflow runs and the small amount of configuration that genuinely differs, while this repository owns the repeated CI/CD implementation.

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
    permissions:
      contents: read
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/lint.yml@v1

  backend:
    permissions:
      contents: read
      id-token: write
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/django-ci.yml@v1
    with:
      django-settings-module: creatorclerk.settings
```

Only jobs that actually authenticate with OIDC should receive `id-token: write`.

The shared workflows should never need individual secret values as inputs. Machine identity IDs and Infisical project IDs are identifiers rather than credentials and may safely be passed as workflow inputs or repository variables.

## OIDC and Infisical

Secret-bearing jobs authenticate to Infisical with GitHub OIDC and short-lived access tokens. The OIDC token remains tied to the **calling repository** even when the implementation comes from a reusable workflow.

A called workflow cannot use OIDC unless the caller grants `id-token: write`; the reusable workflow cannot elevate permissions removed by the caller.

Before a production application is onboarded, its Infisical OIDC binding should constrain the calling repository, event/ref, and `job_workflow_ref` for the approved reusable workflow release. That prevents an application identity from being reused through an arbitrary workflow implementation.

For pull requests from public repositories, secret-bearing jobs must not authenticate for fork PRs. The reusable workflows defensively avoid Infisical authentication for fork pull requests, but callers should also keep untrusted fork jobs secret-free and must not use `pull_request_target` to execute untrusted changes with secrets.

## Deployment security

`deploy-application.yml` is intentionally more restrictive than the generic CI workflows:

- the calling ref must be `refs/heads/deploy`;
- the application name is validated as a safe slug;
- supported caller repositories are explicitly bound to the application they are permitted to deploy;
- overlapping deployments of the same application are serialised;
- the deployment repository is fixed to `adb-software-solutions/adb-deploy@main` rather than being caller-controlled.

The initial repository/application bindings are CreatorClerk, TechWiki, Reseller Workbench, and Wedding of Rebecca and Peter. Additional applications should only be added when their `adb-deploy` application definition and Infisical access model are ready.

The repository/application binding is intentionally explicit for the first release. Longer term, it should be generated or validated from the same application catalogue used by `adb-deploy` so there is only one authoritative mapping.

## Deployment secret mapping

`deploy-application.yml` deliberately avoids application-specific secret maps. Application secrets are read from the application's Infisical project and converted to Ansible extra-vars by convention:

```text
BACKEND_SECRET_KEY -> <application>_backend_secret_key
DB_PASSWORD        -> <application>_db_password
```

Shared platform secrets remain shared, for example `CLOUDFLARE_API_TOKEN -> cloudflare_api_token`.

This keeps application deploy callers small while allowing `vars/applications/<application>.yml` in `adb-deploy` to remain the deployment catalogue.

## Versioning

Feature branches may be used while developing or testing a workflow. Production callers should use a stable release tag such as `@v1`; use an immutable commit SHA where maximum reproducibility and supply-chain protection are required.

Changes to the behaviour of an existing major tag should remain backwards-compatible. Breaking workflow contracts should receive a new major version.

## Public repository policy

The repository is public so both public and private ADB repositories can consume the same reusable workflows. Public visibility is not a security boundary: no credential, private key, password, deployment token, or application secret may ever be committed here.

See [SECURITY.md](SECURITY.md) for the security model and handling expectations.
