# CaseHub Ledger — Session Handover
**Date:** 2026-05-24

## Current State

497 tests, BUILD SUCCESS. Both ledger repos on `main`, pushed to `mdproctor/ledger`
and `casehubio/ledger`. 3 open issues remain.

## What Landed This Session

- **#91** — `casehub-ledger-memory`: 6 `@Alternative @Priority(1)` in-memory stores
  (entries, attestations, trust scores, key rotation, Merkle frontier) + reactive delegates
  gated by `@IfBuildProperty`. Runtime refactors: `LedgerEnricherPipeline` CDI bean
  extracted from `LedgerTraceListener`; `LedgerMerkleFrontierRepository` SPI extracted
  — `LedgerVerificationService` no longer injects `EntityManager`. Hot fix after close:
  `NoOpLedgerMerkleFrontierRepository @DefaultBean` added — consumers (Claudony) were
  failing with `UnsatisfiedResolutionException` because `JpaLedgerMerkleFrontierRepository`
  is `@Alternative` and the injection point had no fallback.

Garden: GE-20260523-de55e8 (CDI proxy field access), GE-20260523-fc1fe7 (Quarkus bytecode
enhancement makes @Entity fields protected in test classloader).
Blog: `blog/2026-05-23-mdp02-persistence-layer-didnt-know.md`

## Immediate Next Step

Pick up one of the 3 remaining open issues. Run `work-start` from `main`.

## What's Left

- `casehubio/parent#62` — ledger deep-dive SPIs table needs `LedgerMerkleFrontierRepository`
  row and `persistence-memory/` module entry · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #85 | External key distribution (TUF/HSM/PKI) for `AgentKeyProvider` | L | High | PQC foundation done; this is the next signing layer |
| #81 | Agent DID/VC cryptographic identity binding | L | High | Depends on #85 implicitly |
| #96 | Code-gen for reactive tier (Vert.x codegen style) | XL | High | Not yet: pair count too small |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-23-mdp02-persistence-layer-didnt-know.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
