# CaseHub Ledger — Session Handover
**Date:** 2026-06-05

## Current State

Both repos on `main`, clean. Branch `issue-116-jpa-sequence-number` closed (EPIC-CLOSED.md). Squashed to 1 commit (`1c3f465`), pushed to both `mdproctor/ledger` and `casehubio/ledger`.

## What Landed This Session

- **#116**: `JpaLedgerEntryRepository.save()` now assigns `sequenceNumber` atomically via `LedgerSequenceAllocator` (SQL-standard MERGE). V1000 adds UNIQUE(subject_id, sequence_number) + `ledger_subject_sequence` table. Also fixed `JpaActorIdentityBindingRepository` — a second persist path that bypassed sequence assignment. 9 tests in `JpaSequenceNumberIT`. Subsumes #100.
- **#120**: Closed (ARC42STORIES.MD completed last session, issue closure + blog publish).
- **#121**: Closed (CLAUDE.md stale `AgentKeyProvider` refs — already fixed in `b1ebe44`).
- **#122**: Filed — PostgreSQL DevServices for real-DB integration tests (H2 `MODE=PostgreSQL` doesn't support `INSERT ON CONFLICT`).
- **Garden**: 5 entries — `GE-20260605-e202fd` (H2 ON CONFLICT rejected), `GE-20260605-159a96` (MERGE KEY resets rows), `GE-20260605-b0b14c` (persist managed entity overwrite), `GE-20260605-b734b3` (MERGE USING portable upsert), `GE-20260605-5d0034` (JOINED inheritance bypass).
- **Blog**: `2026-06-05-mdp01-the-save-that-forgot-to-count.md` + 3 prior entries published.

## Immediate Next Step

Pick from What's Next — `#115` is the largest remaining item but blocked on casehub-engine. `#118` (on-read trust computation) or `#108`/`#110` (identity features, now unblocked) are available.

## What's Left

- Backup branches `backup/pre-squash-main-20260507`, `20260508`, `20260521` — eligible for deletion (past 14-day hold) · XS · Low
- `backup/pre-squash-issue-114-lightweight-mode-20260602` — hold until 2026-06-16 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #115 | Incremental per-actor trust recomputation | L | High | Blocked on casehub-engine `TrustScoreCache.refreshForActor` companion |
| #118 | On-read trust score computation | L | High | Eliminates materialized store staleness |
| #122 | PostgreSQL DevServices for integration tests | M | Med | Validates native SQL + concurrency on real PostgreSQL |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked (~~#81~~ closed) |
| #110 | ScimDIDResolver — synthetic DIDDoc from SCIM | M | Med | Unblocked (~~#107~~ closed) |

## References

| Artifact | Where |
|----------|-------|
| Blog entry | `blog/2026-06-05-mdp01-the-save-that-forgot-to-count.md` |
| Spec (promoted) | `docs/specs/issue-116-jpa-sequence-number/` |
| Garden entries | `GE-20260605-e202fd`, `-159a96`, `-b0b14c`, `-b734b3`, `-5d0034` |
