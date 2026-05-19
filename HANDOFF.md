# CaseHub Ledger — Session Handover
**Date:** 2026-05-20

## Current State

450 tests, BUILD SUCCESS. Both repos on `main`, .m2 installed.
No active branch. All open issues resolved.

## What Landed This Session

**epic-reactive-key-service** (closed):
- #87: debug log in `verifyCryptographic` silent catch
- #82: `LedgerPemUtil` extracted
- #86: `ReactiveKeyRotationService` + `ReactiveLedgerVerificationService` (reactive SPI
  parity); blocking bridge removed from `verifyAgentSignatureAsync`

**issue-92-optional-reactive-repo** (closed):
- #92: Reactive service tier separation — `LedgerVerificationService` and `KeyRotationService`
  stripped of all reactive imports; `ExcludedTypeBuildItem` in `LedgerProcessor` gates reactive
  beans at augmentation via `casehub.ledger.reactive.enabled`; `LedgerBuildTimeConfig`
  in deployment module; `BlockingTierPurityTest` structural enforcement

**Protocols:**
- PP-20260519-f2e160: universal — service beans must not carry optional dependencies
- PP-20260519-39a9a5: casehub — `ExcludedTypeBuildItem` + `@ConfigRoot(BUILD_TIME)` gating
- PP-20260517-15bf75 retired (superseded by above two)

**Garden:** 4 new entries — `isAssignableFrom` direction, reactive SPI shim,
`@ConfigRoot(BUILD_TIME)` SRCFG00050 gotcha, `ExcludedTypeBuildItem` technique

**Protocol gap identified:** `work-end` skill needs a pre-close sweep step before
presenting the artifact inventory — sent to parent for skill update.

## Immediate Next Step

`gh issue list --repo casehubio/ledger --state open` — see current tracker.
Candidates: #85 (external key distribution TUF/HSM), #93 (AgentSignatureVerificationService
split, small refactor), #94 (parity enforcement at compile time).

## Cross-Module

**We're blocking:**
- `casehub-aml`, `casehub-clinical`, `casehub-devtown` — need #92 (`casehub.ledger.reactive.enabled`)
  to build without CDI failures · XS (add one property) · Low

**Not yet blocking, but tracked:**
- `casehub-qhorus#172` — align qhorus with PP-20260519-39a9a5 reactive tier pattern · M · Med

## What's Left

- `issue-92-optional-reactive-repo` workspace branch: retained, deletion due 2026-06-02 · XS · Low
- `epic-reactive-key-service` workspace branch: retained, deletion due 2026-06-02 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #93 | Extract `AgentSignatureVerificationService` — Merkle and signature concerns share `LedgerVerificationService` | S | Low | Follow-on from #92 |
| #85 | External key distribution — TUF/HSM/PKI for `AgentKeyProvider` | M | Med | — |
| #94 | Enforce blocking/reactive parity at compile time | S | Med | Annotation or reflection test approach TBD |
| #81 | Agent DID/VC identity binding | M | High | ADR before code |
| #84 | Post-quantum migration path | M | High | ADR before code |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-19-mdp02-the-wrong-fix-for-the-right-problem.md` |
| New protocols | `~/claude/casehub/parent/docs/protocols/universal/reactive-blocking-tier-separation.md` |
| | `~/claude/casehub/parent/docs/protocols/casehub/reactive-service-build-gating.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
