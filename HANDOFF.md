# CaseHub Ledger — Session Handover
**Date:** 2026-05-21

## Current State

456 tests, BUILD SUCCESS. Both repos on `main`, .m2 installed.
No active branch. All open issues resolved.

## What Landed This Session

**issue-93-agent-signature-verification-service** (closed):
- `AgentCryptographicVerifier` — package-private `final` static utility; Ed25519 verify
  shared by both blocking and reactive tiers; mirrors `LedgerMerkleTree` pattern
- `AgentSignatureVerificationService` — new blocking bean owning `verifyAgentSignature`
- `ReactiveAgentSignatureVerificationService` — renamed from `ReactiveLedgerVerificationService`;
  `verifyCryptographic` duplication eliminated
- `LedgerVerificationService` — stripped to Merkle-only (`treeRoot`, `inclusionProof`, `verify`)
- `BlockingTierPurityTest` extended to cover the new blocking bean
- `LedgerProcessor` updated; design journal merged into `docs/DESIGN.md`

**Garden:** GE-20260520-45312d — `@InjectMock AgentKeyProvider` silent failure in
`@QuarkusTest` when enricher runs at `@PrePersist` with no signing key configured

**Skill bug filed:** `work-end` does not merge project branch to main — leaves
implementation on the branch; one manual `git merge --ff-only` needed after close.
Sent to skills team.

## Immediate Next Step

`gh issue list --repo casehubio/ledger --state open` — see current tracker.
Top candidates: #94 (compile-time parity enforcement, natural follow-on to #93),
#85 (external key distribution TUF/HSM).

## Cross-Module

**Previously blocking (now shipped — consumers need to add property):**
- `casehub-aml`, `casehub-clinical`, `casehub-devtown` — add
  `casehub.ledger.reactive.enabled=false` (or omit, it's the default) to build
  cleanly with #92 changes · XS · Low

**Not yet blocking, but tracked:**
- `casehub-qhorus#172` — align qhorus with PP-20260519-39a9a5 reactive tier pattern · M · Med

## What's Left

- `epic-reactive-key-service` workspace branch: no EPIC-CLOSED.md, deletion overdue · XS · Low
- `issue-92-optional-reactive-repo` workspace branch: no EPIC-CLOSED.md, deletion overdue · XS · Low
- `issue-93-agent-signature-verification-service` workspace branch: deletion due 2026-06-03 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #94 | Enforce blocking/reactive parity at compile time | S | Med | Natural follow-on from #93 |
| #85 | External key distribution — TUF/HSM/PKI for `AgentKeyProvider` | M | Med | — |
| #81 | Agent DID/VC identity binding | M | High | ADR before code |
| #84 | Post-quantum migration path | M | High | ADR before code |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-21-mdp01-the-duplication-that-annotated-itself.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
