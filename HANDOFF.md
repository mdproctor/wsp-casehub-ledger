# CaseHub Ledger — Session Handover
**Date:** 2026-05-15

## Current State

*Unchanged — `git show HEAD~1:HANDOFF.md`* (390 tests, BUILD SUCCESS, clean .m2 install done this session)

## What Happened This Session

**Housekeeping:** Closed stale issues #76, #78, #49, #50, #48 — all work was already done, issues just weren't closed. Ledger is now feature-complete for v0.2 scope with exactly one open issue (#72).

**#72 (cross-repo audit):** casehub-work half confirmed fixed (April 29 session). Claudony half still pending — prompt drafted, waiting for claudony to be free.

**Research:** Web search on ledger/trust field. Five roadmap ideas added to IDEAS.md: bilateral entry signing, agent DID/VC identity, ZK compliance proofs, AntTrust alternative, delegation chain tracking.

**New issue #79 created:** `feat: bilateral entry signing — agent non-repudiation via Ed25519 agent signature`

**Design in progress (brainstorming, mid-session):**
- Key model: **per-actorId** (Option B) — `AgentKeyProvider` SPI with `ConfiguredAgentKeyProvider` default reading from `casehub.ledger.agent.signing.keys.<actorId>`
- Storage: **two nullable columns on `ledger_entry`** — `agent_signature BYTEA`, `agent_public_key BYTEA` (V1005 migration)
- Approach confirmed, design doc not yet written

**ADR 0011** to be written alongside implementation: per-actorId signing key model.

## Key Implementation Constraint

`canonicalBytes()` in `LedgerMerkleTree` is **private**. The `AgentSignatureEnricher` cannot call it directly. Options:
- Expose as `public static canonicalBytes(LedgerEntry)` — preferred
- Duplicate the `subjectId|seqNum|entryType|actorId|actorRole|occurredAt` construction

Decision deferred to design doc / implementation.

## Immediate Next Step

Resume brainstorming: present the full design (components, data flow, verification, testing) and get approval. Then write-plans → implement.

Brainstorming context: Option 1 (columns on `ledger_entry`) approved. Per-actorId SPI key model approved. Next: present complete design covering `AgentKeyProvider` SPI, `AgentSignatureEnricher`, V1005 schema, `LedgerVerificationService.verifyAgentSignature()`.

## Open Issues

- #72 — claudony half pending (prompt drafted — send when claudony is free)
- #79 — bilateral entry signing (in design)
- Federation wiring issue — to be created in parent repo (not yet filed)

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-14-mdp01-quality-at-the-intersection.md` |
| Latest ADRs (project) | `docs/adr/0009-...`, `docs/adr/0010-...` |
| Ideas log | `IDEAS.md` (5 new entries added 2026-05-15) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
