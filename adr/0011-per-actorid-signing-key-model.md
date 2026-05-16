# ADR 0011 — Per-actorId Signing Key Model for Agent Non-Repudiation

**Status:** Accepted
**Date:** 2026-05-15
**Issue:** casehubio/ledger#79

## Context

Bilateral entry signing requires each `LedgerEntry` to be signed by the agent that authored it. The key architectural question is the granularity of signing keys: one key per deployment, or one key pair per actorId.

## Decision

**Per-actorId key pairs.** Each distinct actorId is associated with its own Ed25519 key pair. The `AgentKeyProvider` SPI supplies the appropriate `KeyPair` at signing time. The default implementation (`ConfiguredAgentKeyProvider`) reads PEM file paths keyed by actorId from `casehub.ledger.agent-signing.keys.*`.

## Rationale

**Per-deployment keys are rejected** because they undermine the entire premise of non-repudiation. If all agents share one signing key, any agent (or any actor with access to the deployment) could have produced any signature. The ledger proves the entry was signed, but not *which* agent signed it. This is tamper-evidence, not non-repudiation.

**Per-actorId keys** mean a SOUND attestation from `claude:reviewer@v1` is cryptographically distinguishable from one produced by any other actor. When a key is rotated (see #80), old entries remain verifiable via their stored public key bytes — the self-contained verification model ensures historical entries are never compromised by key rotation.

Industry consensus (Keyfactor, 2025; Aembit, 2025) is that per-agent cryptographic identities are the correct model. Shared deployment credentials create attribution blind spots and dramatically increase blast radius on key compromise.

## Consequences

- Operators configure one key pair per signing agent (private + public PEM files)
- Actors without a configured key pair produce unsigned entries — signing is opt-in
- Key management (generation, rotation, revocation) is out of scope for this issue
- The `AgentKeyProvider` SPI allows consumers to provide per-actorId keys from any source (HSM, Vault, PKI) by replacing the default implementation

## Supersedes

Nothing. First decision in this space.

## Related

- #80 — key rotation pattern
- #81 — agent DID/VC identity binding (the follow-on that adds cryptographic identity verification)
- ADR 0004 — actorId format for LLM agents (`model:persona@major`)
