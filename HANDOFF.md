# CaseHub Ledger — Session Handover
**Date:** 2026-06-09

## Current State

Both repos on `main`, clean. Branch `issue-127-multi-tenancy` closed (EPIC-CLOSED.md). Squashed to 1 commit (`07b493f`), pushed to both `mdproctor/ledger` and `casehubio/ledger`.

## What Landed This Session

- **#127**: Multi-tenancy — `tenancyId` column on `ledger_entry` (V1000), explicit `String tenancyId` on all tenant-scoped SPI methods, `CrossTenantLedgerEntryRepository` split for trust/health/retention jobs, CDI infrastructure (`@CrossTenant`, `@LedgerSystem`, `LedgerSystemCurrentPrincipal`, `CrossTenantProducer`). 86 files changed, 788 tests green.
- **#129**: Filed — `ActorIdentityBindingObserver` needs tenancyId through CDI event chain (interim: defaults to `DEFAULT_TENANT_ID`).
- **Downstream issues filed**: work#260, qhorus#263, engine#459 — SPI propagation.
- **Blog**: `2026-06-09-mdp01-the-tenant-that-was-always-there.md`.

## Immediate Next Step

Pick from What's Next — #108/#110 (identity features) are unblocked. Or start work/qhorus tenancy specs for the cross-repo coherence review.

## What's Left

- Backup branches accumulating — 10+ `backup/pre-squash-*` branches. Oldest eligible for deletion (past 14-day hold): `20260507`, `20260508`, `20260521`, `20260522`. · XS · Low
- parent#181: sync PLATFORM.md and casehub-ledger.md for multi-tenancy (#127) and incremental trust (#115) and PostgreSQL tests (#122). · S · Low
- Cross-repo coherence review: ledger#127 spec + work + qhorus tenancy specs must be reviewed together before downstream implementation begins. · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked |
| #110 | ScimDIDResolver — synthetic DIDDoc from SCIM | M | Med | Unblocked |
| #129 | ActorIdentityBindingObserver — carry tenancyId through CDI event | S | Med | Needs platform-api change |

## References

| Artifact | Where |
|----------|-------|
| Spec (promoted) | `docs/specs/issue-127-multi-tenancy/` |
| Blog entry | `blog/2026-06-09-mdp01-the-tenant-that-was-always-there.md` |
| Downstream issues | work#260, qhorus#263, engine#459 |
