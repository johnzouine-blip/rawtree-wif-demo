# RawTree Workload Identity Federation smoke test

This repository is intentionally minimal. It has one manually triggered GitHub Actions workflow and no application code or dependencies.

## What this first workflow verifies

It requests a GitHub Actions OIDC JWT with the audience `rawtree`, decodes its payload locally, and checks the exact claims intended for the RawTree configuration. It never prints or stores the JWT.

This is the safe first step before enabling token exchange: it proves what GitHub is actually issuing, rather than guessing the subject or audience and risking a 400 `invalid_grant`.

## Expected claims for this repository

For this repository (created after GitHub's July 15, 2026 immutable-subject rollout), a workflow run on the `main` branch without a GitHub Environment should use:

| Claim | Expected value |
|---|---|
| `iss` | `https://token.actions.githubusercontent.com` |
| `aud` | `rawtree` |
| `sub` | `repo:johnzouine-blip@355236458/rawtree-wif-demo@1453687571:ref:refs/heads/main` |
| `ref` | `refs/heads/main` |

The RawTree issuer must use the issuer URL exactly as shown. The federation rule's allowed audience must contain `rawtree`, and its `sub` claim condition must exactly match the subject above. RawTree claim conditions are exact matches.

## Run it

1. In GitHub, open **Actions**.
2. Select **RawTree WIF smoke test**.
3. Select **Run workflow** and choose the `main` branch.
4. Open the run and inspect the **Request GitHub OIDC token and inspect claims** step.

No repository secrets or GitHub Environment are needed. The expected subject includes the immutable owner and repository IDs reported by GitHub. GitHub automatically provides the OIDC request variables to jobs with `id-token: write`.

## Token exchange comes next

This workflow deliberately stops after claim verification. The RawTree token-exchange endpoint and request contract must be taken from the current RawTree API documentation before adding the exchange call. Do not guess the endpoint or grant payload.

## Error triage

- **400 `invalid_grant`**: check issuer, audience, exact claim conditions, signing-key discovery/JWKS, `iat`/`nbf`/`exp`, and whether the assertion has already been consumed. Request a fresh JWT for every exchange attempt.
- **401 on a later API request**: the issued RawTree token may have expired or been revoked.
- **Permission error**: authentication succeeded; inspect the rule's principal, permission/database roles, cluster, and database scope.
