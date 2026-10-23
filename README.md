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
| `frontend-ci.yml` | Install pnpm dependencies, run frontend tests, and build the frontend, optionally against a throwaway MongoDB (`mongo: true`). |
| `docker-build-push.yml` | Build one or more Docker images, authenticate to the ADB registry through Infisical, push, and scan them. Applications now build in `deploy-application.yml` instead. |
| `docker-build-ci.yml` | Build the same images as `docker-build-push.yml` without logging in to the registry or pushing, so pull requests prove their Dockerfiles still build. |
| `deploy-application.yml` | Build the application's images from the deploy commit (optional), then authenticate through OIDC, obtain deployment credentials/secrets from Infisical, and invoke `adb-deploy`. |
| `devcontainer-ci.yml` | Build a repository devcontainer as a CI smoke test. |

## MongoDB in frontend CI

`frontend-ci.yml` with `mongo: true` starts a throwaway MongoDB before tests
and the build:
- it runs `mongo-image` (default `mongo:8.0`) as a single-node replica set,
  so transactions work, as Payload CMS needs;
- it is published on 127.0.0.1 only and has no authentication;
- it exports `DATABASE_URI=mongodb://127.0.0.1:27017/test?replicaSet=rs0`.

`mongo-database` and `mongo-uri-env` change the database and the variable
name. The exported URI takes precedence over the same key from Infisical:
callers that read their prod project on `main` (for build variables) would
otherwise hand tests the production database URI.

```yaml
  frontend:
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/frontend-ci.yml@v1
    with:
      mongo: true
```

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

## Docker build values

`docker-build-push.yml` and `docker-build-ci.yml` can read selected values from an application's Infisical project for frontend/container builds. The `build-arg-keys` and `build-env-keys` mechanisms are intended for **non-sensitive build-time configuration** such as `NEXT_PUBLIC_*` values that happen to be centrally managed in Infisical.

Do not pass passwords, private keys, access tokens, database credentials, or other sensitive values as Docker build arguments or bake them into generated dotenv files. Docker build arguments and files copied into build layers can become visible in image metadata or history. A workload that genuinely needs a sensitive build-time credential should use a BuildKit secret-based mechanism instead.

Temporary build argument and generated dotenv files are removed by the workflow after the build.

`docker-build-ci.yml` takes the same inputs except the registry ones, plus `infisical-environment` (default `dev`). Infisical is optional: without `infisical-identity-id` and `infisical-application-project-id`, and always for fork pull requests, it builds with no build values. For pull requests, pass the PR identity and the CI project, and list build values as optional keys, so a value missing from `dev` doesn't fail the build:

```yaml
docker-build:
    if: ${{ github.event_name == 'pull_request' && github.event.pull_request.draft == false }}
    permissions:
        contents: read
        id-token: write
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/docker-build-ci.yml@v1
    with:
        application: example
        images: '["website"]'
        infisical-identity-id: <PR identity>
        infisical-application-project-id: <CI project>
        build-env-file: website/.ci.env
        optional-build-env-keys: '["NEXT_PUBLIC_API_URL"]'
```

## Build on deploy

Applications build their images in `deploy-application.yml`, from the commit
on `deploy`, and deploy that commit's SHA in the same run. What is live is
therefore exactly the `deploy` branch: a newer `main` build can never be
deployed by accident, and a deploy never races a still-running `main` build.

```yaml
on:
  push:
    branches: [deploy]
  workflow_dispatch:
    inputs:
      image_tag:
        description: Leave empty to build and deploy this commit; a full Git SHA redeploys that image (rollback)
        type: string
        default: ""

jobs:
  deploy:
    uses: adb-software-solutions/github-actions-workflows/.github/workflows/deploy-application.yml@v1
    with:
      application: stackedfinds
      image-tag: ${{ inputs.image_tag }}
      infisical-registry-project-id: <adb-container-registry project>
      build-images: >-
        ["backend", "celery-worker", "celery-beat",
         {"image": "website", "build-env-file": "website/.ci.env",
          "build-env-keys": ["NEXT_PUBLIC_API_URL", "NEXT_PUBLIC_SITE_URL"]}]
      # ...plus the existing infisical-* deploy inputs
```

- `image-tag` empty: build `build-images` and deploy `${GITHUB_SHA}`.
- `image-tag` set: deploy that image without building (redeploy, rollback).
- No `build-images`: deploy `latest`, as before (callers not yet migrated).

Each `build-images` entry is an image suffix or an object with `image` and
any of `context`, `build-env-file`, `build-env-keys`,
`optional-build-env-keys`, `build-arg-keys` and `optional-build-arg-keys`;
the workflow inputs of the same names are the defaults. Images build in
parallel, and the deploy runs only if every build succeeded.

The build job runs inside `deploy-application.yml`, so it uses the
application's **deploy** identity, whose OIDC binding only accepts this
workflow on `refs/heads/deploy`; that identity already reads the registry and
application projects. The build steps are the composite action
`.github/actions/docker-build-push`, shared with `docker-build-push.yml`.
`main` keeps its lint and tests; pull requests still prove the Dockerfiles
build with `docker-build-ci.yml`.

## Deployment security

`deploy-application.yml` is intentionally more restrictive than the generic CI workflows:

- the calling ref must be `refs/heads/deploy`;
- the application name is validated as a safe slug;
- supported caller repositories are explicitly bound to the application they are permitted to deploy;
- overlapping deployments of the same application are serialised;
- the deployment repository is fixed to `adb-software-solutions/adb-deploy@main` rather than being caller-controlled.

The initial repository/application bindings are CreatorClerk, TechWiki, Reseller Workbench, and Wedding of Rebecca and Peter. Additional applications should only be added when their `adb-deploy` application definition and Infisical access model are ready.

The repository/application binding is intentionally explicit for the first release. Longer term, it should be generated or validated from the same application catalogue used by `adb-deploy` so there is only one authoritative mapping.

## Deployment targets

The target host is the application's `host` in its `adb-deploy` definition (or
`application_host_override` passed in `ansible-extra-vars-json`).

Every host is built by `adb-iac` (the legacy port-1999 hosts and their
shared `DEPLOY_SSH_KEY` were retired in October 2026):

- **Hosts** are resolved from the adb-iac deploy inventory
  (`adb-deploy/scripts/fetch-iac-inventory.sh`, read-only credentials from
  `adb-deployment-bootstrap /r2`). The job then:
  - joins the tailnet through Tailscale workload identity federation, using
    the host's deploy pool credential. That credential is bound to the
    callers allowed to deploy to the pool, so other repositories cannot
    obtain the pool's runner tag;
  - connects as `deploy` on port 22 with the pool's own key
    (`IAC_DEPLOY_SSH_KEY_<POOL>`), trusting only the published, pinned host
    keys (no `ssh-keyscan`);
  - refuses hosts the inventory lists as pending, and shows the reason.

See adb-iac `docs/DEPLOY-INVENTORY.md` for pools and the inventory format.

## Deployment secret mapping

`deploy-application.yml` deliberately avoids application-specific secret maps. Application secrets are read from the application's Infisical project and converted to Ansible extra-vars by convention:

```text
BACKEND_SECRET_KEY -> <application>_backend_secret_key
DB_PASSWORD        -> <application>_db_password
```

Shared platform secrets remain shared, for example `CLOUDFLARE_API_TOKEN -> cloudflare_api_token`.

The application-backup R2 credentials in `adb-infrastructure` `/cloudflare` are passed too, when present: `R2_APP_BACKUP_ACCESS_KEY_ID -> app_backup_r2_access_key_id` and `R2_APP_BACKUP_SECRET_ACCESS_KEY -> app_backup_r2_secret_access_key`. They are optional here; adb-deploy refuses to deploy an application with data to back up without them.

This keeps application deploy callers small while allowing `vars/applications/<application>.yml` in `adb-deploy` to remain the deployment catalogue.

## Versioning

Feature branches may be used while developing or testing a workflow. Production callers should use a stable release tag such as `@v1`; use an immutable commit SHA where maximum reproducibility and supply-chain protection are required.

Changes to the behaviour of an existing major tag should remain backwards-compatible. Breaking workflow contracts should receive a new major version.

## Public repository policy

The repository is public so both public and private ADB repositories can consume the same reusable workflows. Public visibility is not a security boundary: no credential, private key, password, deployment token, or application secret may ever be committed here.

See [SECURITY.md](SECURITY.md) for the security model and handling expectations.
