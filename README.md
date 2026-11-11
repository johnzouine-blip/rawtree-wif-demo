# RawTree Workload Identity Federation smoke test

This repository contains one manually triggered GitHub Actions workflow. It requests a GitHub OIDC assertion, exchanges it against the existing RawTree federation rule, and runs a harmless read-only query.

## Existing RawTree rule

The workflow is designed to use the **same rule already created in RawTree**. It does not create or update federation rules.

For a run on `main` without a GitHub Environment, the expected GitHub claims are:

| Claim | Expected value |
|---|---|
| `iss` | `https://token.actions.githubusercontent.com` |
| `aud` | `rawtree` |
| `sub` | `repo:johnzouine-blip@355236458/rawtree-wif-demo@1453687571:ref:refs/heads/main` |
| `ref` | `refs/heads/main` |

The existing RawTree rule must match this issuer, audience, and exact `sub` value, map to your selected RawTree principal, target the cluster containing `analytics`, and be enabled. The workflow does not configure a GitHub Environment because that would change the subject claim and no longer match the current branch-based rule.

For this query-only test, the rule should have `read_only` permission and database `analytics`. If the rule currently has a different `sub` condition, do not create a second rule; update the existing rule only if you intend to keep using this repository and branch as its trust boundary.

## GitHub repository variables

In the repository, open **Settings → Secrets and variables → Actions → Variables** and add these three **repository variables** (not secrets):

| Variable | Value |
|---|---|
| `RAWTREE_URL` | The API URL of the RawTree deployment serving the cluster with this rule |
| `RAWTREE_FEDERATION_RULE_ID` | The ID of the existing federation rule above |
| `RAWTREE_AUDIENCE` | `rawtree` |

Do not add an API key or any long-lived RawTree credential. The workflow obtains its GitHub assertion at runtime and exchanges it for a short-lived RawTree bearer token.

## What the workflow does

1. Requests a fresh GitHub OIDC assertion for audience `rawtree`.
2. Sends it to `POST $RAWTREE_URL/oauth/token` using the JWT-bearer grant and the configured existing `federation_rule_id`.
3. Extracts the returned bearer token without printing it.
4. Sends `POST $RAWTREE_URL/v1/query` with `{"sql":"SELECT 42 AS value"}` and the `x-rawtree-database: analytics` header.
5. Passes only if RawTree returns `value = 42`.

RawTree workload tokens last at most 15 minutes and never outlive the provider assertion. Each assertion can be exchanged only once per issuer configuration. The workflow requests a fresh assertion each time it runs; retry a failed exchange with a new workflow run rather than reusing an assertion.

## Run it

1. In GitHub, open **Actions**.
2. Select **RawTree WIF smoke test**.
3. Select **Run workflow** and choose the `main` branch.
4. Inspect the job log. It should report that RawTree issued a bearer token (with its value redacted), followed by the query PASS message.

The workflow has `id-token: write`, which allows GitHub Actions to request the OIDC assertion. It does not need repository contents write permission, a GitHub Environment, repository secrets, or a RawTree API key.

## Error triage

- **400 `invalid_grant`**: check issuer, audience, required claims, signing-key discovery/JWKS, expiry, exact rule conditions, and whether the assertion was already used. Run the workflow again to obtain a fresh assertion.
- **401 on the query request**: the RawTree token may have expired or been revoked, or the rule/issuer/principal may have changed.
- **Permission error**: authentication succeeded, but the rule or database permissions do not allow the query. Confirm the rule uses `read_only` (or a suitable existing database role) for `analytics`.
- **Missing repository variable**: add the named variable under **Settings → Secrets and variables → Actions → Variables** and rerun.
