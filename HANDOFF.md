# CaseHub Ledger — Session Handover
**Date:** 2026-05-23

## Current State

452 tests, BUILD SUCCESS. Both ledger repos on `main`, pushed to `mdproctor/ledger`
and `casehubio/ledger`. 4 open issues remain.

## What Landed This Session

- **#90** — Closed won't-do: moving `LedgerTraceIdProvider` to `platform-api` reduces
  one import but changes nothing about the classpath; the coupling was never what it looked like.
- **#97** — `checkUniTypeArg()` in `haveBidirectionalMethodParity()`: verifies `Uni<T>`
  type argument erasure matches blocking return type. 2 synthetic meta-tests.
- **#84** — Algorithm-transparent signing: `AgentSignatureEnricher`, `AgentCryptographicVerifier`,
  `LedgerMerklePublisher`, `LedgerPemUtil` no longer hardcode `"Ed25519"`. ADR 0013.
  Code review caught `LedgerPemUtil` — without that fix, the ADR's "no code changes at adoption
  time" claim was false.

Garden: GE-20260523-7fcea7 (ArchUnit isEmpty() dead code), GE-20260523-0ecc24 (JCA trial-load
technique), GE-20260523-722840 (ArchUnit meta-test with evaluate().hasViolation()).
Protocol: PP-20260523-e7b577 (ledger-algorithm-transparent-signing).
Blog: `blog/2026-05-23-mdp01-the-missing-pem-loader.md`

## Immediate Next Step

Pick up one of the 4 remaining open issues. Run `work-start` from `main`.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #91 | `persistence-memory/` module — in-memory repo impls, zero-config ephemeral install | M | Med | Store SPI pattern; needed by eidos too |
| #90 | Evaluate `LedgerTraceIdProvider` → platform-api | — | — | Closed won't-do this session |
| #85 | External key distribution (TUF/HSM/PKI) for `AgentKeyProvider` | L | High | PQC foundation done; this is the next signing layer |
| #81 | Agent DID/VC cryptographic identity binding | L | High | Depends on #85 implicitly |
| #96 | Code-gen for reactive tier (Vert.x codegen style) | XL | High | Not yet: pair count (2) is too small |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-23-mdp01-the-missing-pem-loader.md` |
| ADR 0013 | `docs/adr/0013-post-quantum-signing-migration.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
