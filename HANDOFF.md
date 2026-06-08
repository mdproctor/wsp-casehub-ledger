# CaseHub Ledger — Session Handover
**Date:** 2026-06-08

## Current State

Both repos on `main`, clean. Branch `issue-122-postgresql-devservices` closed (EPIC-CLOSED.md). Squashed to 1 commit (`cc44ea9`), pushed to both `mdproctor/ledger` and `casehubio/ledger`.

## What Landed This Session

- **#122**: PostgreSQL Testcontainers for integration tests. `PostgreSQLTestResource` (Testcontainers lifecycle) + `PostgreSQLTestProfile` (abstract base). Build-time `db-kind` override via `getConfigOverrides()` (triggers re-augmentation); runtime JDBC config via test resource. `JpaSequenceNumberPgIT` validates MERGE INTO on PostgreSQL 17. `LedgerHealthJobPgIT` validates JPQL aggregation. Case-insensitive constraint assertion fix for cross-DB portability.
- **Garden**: REVISE GE-20260601-b76fba — alternative solution for per-profile build-time overrides via `getConfigOverrides()`.
- **Blog**: `2026-06-08-mdp01-the-property-that-wouldnt-move.md`.

## Immediate Next Step

Pick from What's Next — #108/#110 (identity features) are unblocked.

## What's Left

- Backup branches accumulating — 10+ `backup/pre-squash-*` branches. Oldest eligible for deletion (past 14-day hold): `20260507`, `20260508`, `20260521`, `20260522`. · XS · Low
- parent#181: sync PLATFORM.md and casehub-ledger.md for incremental trust (#115) and PostgreSQL tests (#122). · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked |
| #110 | ScimDIDResolver — synthetic DIDDoc from SCIM | M | Med | Unblocked |

## References

| Artifact | Where |
|----------|-------|
| Blog entry | `blog/2026-06-08-mdp01-the-property-that-wouldnt-move.md` |
| Spec (promoted) | `docs/specs/issue-122-postgresql-devservices/` |
| Garden revision | `GE-20260601-b76fba` |
