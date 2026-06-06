# CaseHub Ledger — Session Handover
**Date:** 2026-06-06

## Current State

Both repos on `main`, clean. Branch `issue-115-incremental-trust-recomputation` closed (EPIC-CLOSED.md). Squashed to 1 commit (`a94815f`), pushed to both `mdproctor/ledger` and `casehubio/ledger`.

## What Landed This Session

- **#115**: Incremental per-actor trust recomputation. `IncrementalTrustUpdateObserver` (`AFTER_SUCCESS` + `REQUIRES_NEW`) triggers per-actor Bayesian Beta recomputation when an attestation is saved. `PerActorTrustComputer` extracted from `TrustScoreJob` — shared by batch and incremental paths. `TrustScoreActorUpdatedEvent` for downstream consumers. Gated by `casehub.ledger.trust-score.incremental.enabled` (default false). Batch job remains as backstop.
- **parent#181**: Filed — sync PLATFORM.md and casehub-ledger.md for incremental trust.
- **Garden**: `GE-20260606-0c9216` — `@QuarkusTest @Transactional` defers AFTER_SUCCESS CDI observers.
- **Blog**: `2026-06-06-mdp01-the-observer-that-couldnt-wait.md`.

## Immediate Next Step

Pick from What's Next — #118 (on-read trust) or #122 (PostgreSQL DevServices) are the most impactful. #108/#110 (identity features) are unblocked.

## What's Left

- Backup branches accumulating — 10 `backup/pre-squash-*` branches. Oldest eligible for deletion (past 14-day hold): `20260507`, `20260508`, `20260521`, `20260522`. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #118 | On-read trust score computation | L | High | Eliminates materialized store staleness |
| #122 | PostgreSQL DevServices for integration tests | M | Med | Validates native SQL + concurrency on real PostgreSQL |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked |
| #110 | ScimDIDResolver — synthetic DIDDoc from SCIM | M | Med | Unblocked |

## References

| Artifact | Where |
|----------|-------|
| Blog entry | `blog/2026-06-06-mdp01-the-observer-that-couldnt-wait.md` |
| Spec (promoted) | `docs/specs/issue-115-incremental-trust-recomputation/` |
| Garden entry | `GE-20260606-0c9216` |
