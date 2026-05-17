# ADR 0012 — Agent Signing Key Rotation: Self-Derived keyRef and First-Class Rotation Events

**Status:** Accepted
**Date:** 2026-05-17
**Issue:** casehubio/ledger#80

## Context

Bilateral entry signing (#79) stores raw Ed25519 public key bytes alongside each signed `LedgerEntry`. This enables self-contained verification. What was missing: key identity attribution (which generation signed this entry?), rotation records, and compromise detection.

## Decisions

### 1. Self-derived `keyRef`

`keyRef = Base64URL(SHA-256(publicKey.getEncoded()))` — derived entirely from the public key bytes.

**Rejected:** operator-configured key IDs (misconfiguration risk, coordination required across deployments), sequence counters (requires stateful ledger bookkeeping).

**Rationale:** Aligns with Sigstore/Rekor v2's key identity model. Zero operator configuration. Any party with the public key can independently compute and verify the keyRef. Old entries can retroactively derive their keyRef from stored `agentPublicKey` bytes, enabling backfill without re-signing.

### 2. `KeyRotationEntry` as first-class `LedgerEntry` subclass

Rotation events are immutable ledger entries in the tamper-evident chain, not a separate audit table.

**Rejected:** separate `ledger_key_rotation` table (no Merkle participation, no sequence), `KeyRotationSupplement` (blurs purpose of attached entry).

**Rationale:** Key rotation is an observable event in the actor's lifecycle. It belongs in the chain. The `subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` derivation makes the full key lifecycle queryable as an ordered ledger sequence per actor.

### 3. NIST SP 800-57 distinction: SCHEDULED vs COMPROMISED

Two distinct `KeyRotationReason` values, not a single "rotate" event.

**Rationale:** NIST SP 800-57 explicitly separates rotation (planned, cryptoperiod-driven) from revocation (compromise response). Only COMPROMISED records with `effectiveSince` produce SUSPECT entries in `verifyAgentSignature`. SCHEDULED rotations never retroactively affect existing entries.

### 4. `VerificationResult.SUSPECT` — not `INVALID`

A cryptographically valid signature from a subsequently compromised key returns SUSPECT, not INVALID.

**Rationale:** The entry content is intact and was genuinely signed by that actor at that time. INVALID would be misleading — it implies tampering. SUSPECT correctly conveys "the signature is authentic but the signing actor's trustworthiness is in question."

## Consequences

- `AgentKeyProvider.signingKey()` returns `Optional<SigningKey>` (was `Optional<KeyPair>`)
- `LedgerEntry` gains `agentKeyRef TEXT` field (V1006 migration)
- `KeyRotationEntry` subclass table (V1007 migration)
- `LedgerVerificationService.verifyAgentSignature()` queries compromise windows on VALID signatures

## Related

- #79 — bilateral entry signing (foundation)
- #83 — CDI event on SUSPECT detection
- #84 — post-quantum algorithm migration
- #85 — external key distribution (TUF/HSM/PKI)
- ADR 0011 — per-actorId signing key model
