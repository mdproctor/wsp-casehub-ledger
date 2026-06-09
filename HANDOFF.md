# CaseHub Ledger — Session Handover
**Date:** 2026-06-10

## Current State

Both repos on `main`, clean. Branch `issue-131-tenancy-field-shadowing` closed (EPIC-CLOSED.md). Squashed to 1 commit (`b68f125`), pushed to both `mdproctor/ledger` and `casehubio/ledger`.

## What Landed This Session

- **#131 + #132**: `@CrossTenant` re-scoped to dual-variant disambiguation only (Category 1: `CrossTenantLedgerEntryRepository`). Category 2 repos (`ActorTrustScoreRepository`, `KeyRotationRepository`, `ActorIdentityBindingRepository`) stay unqualified — type enforces boundary. `CrossTenantProducer` deleted. `@LedgerSystem` + `LedgerSystemCurrentPrincipal` deleted (dead code). Two `@BuildStep` guards in `LedgerProcessor`: field-shadowing detection (Jandex ancestor chain walk) and `@CrossTenant` scope validation (`@RequestScoped` rejected). 23 files changed, 712 tests green.
- **Protocol updated**: `ledger-subclass-extension.md` — field-shadowing rule + build-time enforcement.
- Engine-side changes (#460) deferred — engine repo in use by another session.

## Immediate Next Step

Pick from What's Next — #108/#110 (identity features) are unblocked. Or apply engine#460 (field shadowing fix) when engine repo is available.

## What's Left

- Backup branches accumulating — oldest eligible past 14-day hold: `20260507`, `20260508`, `20260521`, `20260522`. · XS · Low
- parent#181: sync PLATFORM.md for multi-tenancy (#127), incremental trust (#115), PostgreSQL tests (#122). · S · Low
- Cross-repo coherence review: ledger#127 spec + work + qhorus tenancy specs. · M · Med
- engine#460: remove tenancyId field from CaseLedgerEntry/WorkerDecisionEntry, capture code cleanup, V2000/V2001 migration rewrite. Spec documents exact changes. · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked |
| #110 | ScimDIDResolver — synthetic DIDDoc from SCIM | M | Med | Unblocked |

## References

| Artifact | Where |
|----------|-------|
| Spec (promoted) | `docs/specs/issue-131-tenancy-field-shadowing/` |
| Blog entry | `blog/2026-06-09-mdp01-the-tenant-that-was-always-there.md` |
| Protocol update | `garden/docs/protocols/casehub/ledger-subclass-extension.md` |
