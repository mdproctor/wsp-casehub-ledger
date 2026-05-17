# CaseHub Ledger — Session Handover
**Date:** 2026-05-17

## Current State

432 tests, BUILD SUCCESS. Both repos on `main`. No active epic. All open bugs resolved.

## What Landed This Session

**#79** (bilateral entry signing) and **#80** (key rotation) — both shipped and pushed to main.

Key rotation highlights:
- `SigningKey` record: self-derived `keyRef = Base64URL(SHA-256(pubKey))` — zero operator config, Sigstore-aligned
- `KeyRotationReason`: SCHEDULED | COMPROMISED (NIST SP 800-57 distinction)
- `KeyRotationEntry`: first-class `LedgerEntry` subclass; subjectId derived from actorId
- `VerificationResult.SUSPECT`: valid signature, compromised key — distinct from INVALID
- ADRs 0011 (per-actorId key model) and 0012 (key rotation design)

**#83** (AgentSignatureSuspectEvent) — shipped and epic closed.
- `AgentSignatureSuspectEvent` CDI record; consumers choose @Observes or @ObservesAsync
- `verifyAgentSignature()` fires `event.fire()` on SUSPECT
- New `verifyAgentSignatureAsync(UUID)` — reactive twin, `Uni<VerificationResult>`, fires `event.fireAsync()`
- Blocking bridge for `compromisedWindows()` pending #86 (reactive KeyRotationService)

**Epic skill improved:** empty journal at close now surfaces `[W]rite / [S]kip` instead of silently skipping.

**New protocol:** PP-20260517-15bf75 — `ledger-sync-async-parity`: all new ledger service methods must ship both blocking and reactive variants.

## Open Issues

| # | What | Notes |
|---|------|-------|
| #81 | Agent DID/VC identity binding | Research horizon |
| #82 | Extract LedgerPemUtil | Small refactor, any time |
| #83 | Closed ✅ | |
| #84 | Post-quantum migration | Research |
| #85 | External key distribution (TUF/HSM) | Medium |
| #86 | Reactive KeyRotationService | Blocks async path bridge removal |
| #87 | Debug log in verifyCryptographic catch | Minor |

## Immediate Next Step

`gh issue list --repo casehubio/ledger --state open` to see current tracker. Next candidates: **#82** (quick PEM utility refactor) or **#86** (reactive KeyRotationService — follows PP-20260517-15bf75).

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-17-mdp01-trust-evidence-architecture.md` |
| Latest ADRs | `adr/0011-per-actorid-signing-key-model.md`, `adr/0012-key-rotation-design.md` |
| New protocol | `~/claude/casehub/parent/docs/protocols/casehub/ledger-sync-async-parity.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
