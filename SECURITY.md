# Security

This repository is intentionally public. It contains reusable GitHub Actions workflow definitions only and must never contain secret values, private keys, access tokens, passwords, or other credentials.

## Security model

Sensitive workflows authenticate to Infisical with GitHub OIDC. The machine identity and project IDs passed to a workflow are identifiers, not credentials. Infisical remains responsible for authorising the calling repository, event, ref, and approved reusable workflow claims before issuing a short-lived access token.

Application repositories must grant only the permissions required by the called workflow. Jobs that use Infisical require `contents: read` and `id-token: write`; workflows that do not use OIDC should not request `id-token: write`.

Production callers should reference a stable release tag or immutable commit SHA. Application-specific secret values must remain in Infisical and must not be passed as `workflow_call` inputs.

Pull-request jobs that execute untrusted fork code must not authenticate to Infisical. The reusable frontend and CI workflows include a defensive fork check, but callers are also responsible for keeping fork jobs secret-free and must not use `pull_request_target` to execute untrusted changes with secrets.

## Deployment trust

Deployment is restricted to callers running from `refs/heads/deploy`. Before production application onboarding, the Infisical OIDC bindings should also constrain `job_workflow_ref` to the approved reusable workflow release so an application deployment identity cannot be used through an arbitrary workflow implementation.

## Reporting

If a credential is ever committed here, treat it as compromised: revoke or rotate it immediately, remove it from active use, and then clean up the repository history as a separate step.
