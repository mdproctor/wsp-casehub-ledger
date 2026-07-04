# Session Handoff — 2026-07-04

## Branch closed: issue-170-vault-oidc-auth

Added JWT auth method to Vault Transit signing adapter (#170):
- `JwtVaultTokenSource` with `Supplier<String>` for JWT acquisition
- `KubernetesVaultTokenSource` consolidated into `JwtVaultTokenSource.fromFile()`
  with `mountPath="kubernetes"` — identical loginPath/loginRequestBody semantics
- `AuthMethod.JWT` added to Quarkus config; `jwtPath` changed to `Optional<String>`
  with auth-method-specific defaults; `jwt()` property for static JWT strings
- Design review (2 rounds, 8 issues, 6 verified, 2 accepted) caught the consolidation

Also filed #171 (browser-based OIDC — deferred, separate use case).

## Current state

- `casehubio/ledger` main: `1b392e9` — pushed
- Issues #170 closed; #171 filed

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm. No consumer exists yet.

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #171 | Browser-based Vault OIDC flow (two-step auth URL + callback) | M | Med | Filed this session; not needed until interactive admin tooling |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | paused — waiting for casehub-ops consumer |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count >= 5 |
