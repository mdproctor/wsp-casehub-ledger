# Session Handoff — 2026-06-28

## Branch closed: issue-159-normalize-dual-channel-cdi

Normalized all CDI event producers to dual-channel firing (#159). Every producer now fires both `fire()` and `fireAsync()`. Fixed two latent bugs: `ReactiveKeyRotationService.fireAsync()` silently skipped sync cache-invalidation observers; `ReactiveAgentSignatureVerificationService` coupled the SUSPECT verdict to observer delivery success. Garden entry GE-20260628-3ea24f captures the Mutiny/fireAsync coupling gotcha.

## Current state

- `casehubio/ledger` main: `0d92bb9` — pushed to origin
- All 6 modules BUILD SUCCESS
- Issue #159 closed

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm (first clarifying question asked: per-hash vs lineage identity model). Resume with `/work`.

## Immediate Next Step

Resume #137 brainstorm, or pick from the backlog.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | paused — brainstorm in progress |
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | blocker #85 closed — ready |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | blocker #85 closed — ready |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count >= 5 |
