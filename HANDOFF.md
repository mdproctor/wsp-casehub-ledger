# Session Handoff — 2026-07-03

## Branch closed: issue-101-vault-approle-oidc-auth

Added pluggable Vault auth to VaultTransitAgentSigner (#101):
- VaultTokenSource SPI with StaticVaultTokenSource, AppRoleVaultTokenSource,
  KubernetesVaultTokenSource — lazy renewal, clamped expiry buffer
- Signing client now stateless (token as per-call parameter)
- 403-retry in Quarkus adapter via tokenSource.invalidate() + single retry
- VaultAuthenticationException for type-safe auth failure detection

Also fixed #169 (ScimActorDIDProviderIT CDI qualifier mismatch) on the same branch.

Design review: 10 rounds, 14 issues raised, 11 verified, 2 accepted, 1 deferred.
Garden entry: GE-20260701-82f303 (CompositeActorDIDProvider instanceof gotcha).

## Current state

- `casehubio/ledger` main: `d1948e4` — pushed
- Issues #101, #169 closed; #170 filed (actual OIDC auth, out of scope)

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm. No consumer exists yet.

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | paused — waiting for casehub-ops consumer |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count >= 5 |
