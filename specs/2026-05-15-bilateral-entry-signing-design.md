# Bilateral Entry Signing — Design Spec
**Issue:** casehubio/ledger#79
**Date:** 2026-05-15
**Status:** Approved for implementation

---

## Problem

`LedgerEntry` records are signed by the ledger (Merkle MMR + Ed25519 checkpoint), proving entries haven't been tampered with. But the *agent* that produced an entry never signs it. An agent can later deny authoring an action — the ledger proves the entry exists and hasn't changed, but not that the specific agent wrote it.

Non-repudiation requires the agent's own signature. EU AI Act Article 12 requires reconstructing "who authorised this" — the ledger's signature alone cannot provide that.

---

## Design

### Core decision: self-contained verification

Public key bytes are stored alongside the signature on each entry. Verification requires no external key management system — entries remain independently verifiable in perpetuity. This is intentional for an immutable audit ledger: liveness of a key store must never be a prerequisite for verifying a historical record.

Key rotation (#80) and DID/VC identity binding (#81) are follow-on issues. This design provides the foundation without foreclosing either.

---

### Schema — V1005

```sql
ALTER TABLE ledger_entry
    ADD COLUMN agent_signature  BYTEA,
    ADD COLUMN agent_public_key BYTEA;
```

Both nullable. Unsigned entries carry nulls — no overhead, no constraint.

---

### Model — `LedgerEntry`

```java
@Column(name = "agent_signature")
public byte[] agentSignature;

@Column(name = "agent_public_key")
public byte[] agentPublicKey;
```

---

### `AgentKeyProvider` SPI

```java
public interface AgentKeyProvider {
    /**
     * Returns the Ed25519 private key for the given actorId, or empty
     * if this actor does not participate in bilateral signing.
     */
    Optional<KeyPair> signingKeyPair(String actorId);
}
```

Returns `KeyPair` (not just `PrivateKey`) so the enricher can store the public key bytes without a separate derivation step.

**Default:** `NoOpAgentKeyProvider` (`@DefaultBean`) — returns empty for all actors. Signing is opt-in.

**Configurable default:** `ConfiguredAgentKeyProvider` (`@ApplicationScoped`, activated when config is present) — reads PKCS#8 PEM private key paths from `casehub.ledger.agent.signing.keys.<actorId>`. Public key is derived from the private key via `KeyFactory` — same approach as `LedgerMerklePublisher.loadPrivateKey()`. Multiple actors configurable.

```
casehub.ledger.agent.signing.keys."claude:reviewer@v1"=/secrets/reviewer-v1.pem
casehub.ledger.agent.signing.keys."claude:auditor@v1"=/secrets/auditor-v1.pem
```

---

### `LedgerMerkleTree.canonicalBytes()` — visibility change

`canonicalBytes(LedgerEntry)` promoted from `private` to `public static`. It is the authoritative canonical form for both Merkle leaf hashes and agent signatures — two consumers, one implementation.

The canonical form is unchanged:
```
subjectId|sequenceNumber|entryType|actorId|actorRole|occurredAt(millis)
```

---

### `AgentSignatureEnricher` — `LedgerEntryEnricher` implementation

Runs in the `@PrePersist` enricher pipeline. Non-fatal — failures are logged, entry still persists unsigned.

```java
@ApplicationScoped
public class AgentSignatureEnricher implements LedgerEntryEnricher {

    @Inject AgentKeyProvider keyProvider;

    @Override
    public void enrich(LedgerEntry entry) {
        if (entry.actorId == null) return;
        keyProvider.signingKeyPair(entry.actorId).ifPresent(keyPair -> {
            byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
            entry.agentSignature = sign(canonical, keyPair.getPrivate());
            entry.agentPublicKey = keyPair.getPublic().getEncoded();
        });
    }
}
```

Signing reuses `LedgerMerklePublisher.signCheckpoint()` logic (static Ed25519 via `java.security.Signature`).

---

### `VerificationResult` — new enum

```java
public enum VerificationResult {
    /** No agent signature stored — actor did not sign. */
    UNSIGNED,
    /** Signature present and valid against stored public key and canonical bytes. */
    VALID,
    /** Signature present but verification failed — possible tampering. */
    INVALID
}
```

---

### `LedgerVerificationService.verifyAgentSignature(UUID entryId)`

```java
@Transactional
public VerificationResult verifyAgentSignature(UUID entryId) {
    LedgerEntry entry = ledgerRepo.findEntryById(entryId)
        .orElseThrow(() -> new IllegalArgumentException("Entry not found: " + entryId));

    if (entry.agentSignature == null) return VerificationResult.UNSIGNED;

    byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
    return verify(canonical, entry.agentSignature, entry.agentPublicKey)
        ? VerificationResult.VALID
        : VerificationResult.INVALID;
}
```

---

## What's out of scope

| Concern | Tracked |
|---------|---------|
| Key rotation pattern | #80 |
| Agent DID/VC identity binding | #81 |
| REST/MCP endpoint to expose `verifyAgentSignature` | Consumers add their own endpoints |
| Mandatory signing enforcement | Opt-in by design; mandate is a consumer policy |

---

## Testing strategy

**Unit — `AgentSignatureEnricherTest`**
- Signs an entry with a generated test key pair; asserts `agentSignature` and `agentPublicKey` populated
- No-op when `AgentKeyProvider` returns empty; asserts fields remain null
- Idempotent: enrich called twice does not corrupt the signature

**Unit — `LedgerVerificationServiceTest`**
- UNSIGNED: entry with null `agentSignature` → `UNSIGNED`
- VALID: entry signed with known key pair → `VALID`
- INVALID: `agentSignature` tampered → `INVALID`
- INVALID: `actorId` mutated after signing → `INVALID` (canonical form includes actorId)

**Integration — `AgentSigningIT` (QuarkusTest)**
- Persists a signed entry end-to-end via `ConfiguredAgentKeyProvider` with a test PEM
- Calls `verifyAgentSignature` → asserts `VALID`
- Mutates `agentSignature` bytes in DB directly → asserts `INVALID`
- Persists an entry with unconfigured actor → asserts `UNSIGNED`

**ADR 0011** — per-actorId signing key model: written and committed alongside implementation.
