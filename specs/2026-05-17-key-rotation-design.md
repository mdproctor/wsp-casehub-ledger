# Agent Signing Key Rotation — Design Spec
**Issue:** casehubio/ledger#80
**Date:** 2026-05-17
**Status:** Approved for implementation

---

## Problem

Bilateral entry signing (#79) stores raw Ed25519 public key bytes alongside each signed `LedgerEntry` for self-contained verification. This is correct — old entries remain verifiable forever from their stored bytes after a key rotation. What's missing:

1. **No key attribution** — entries carry public key bytes but no identifier linking them to a named key generation. An auditor cannot efficiently query "which entries did key generation X sign?"
2. **No rotation record** — there is no immutable record of when a key was rotated, why, or what key it replaced.
3. **No compromise detection** — when a key is reported compromised, there is no mechanism to identify which entries are now suspect.

---

## Design

### Core decision: self-derived `keyRef`

The `keyRef` is `Base64URL(SHA-256(publicKey.getEncoded()))` — derived entirely from the public key bytes. Zero operator configuration. Any party with the public key can verify the keyRef. Old entries can retroactively derive their keyRef from their stored `agentPublicKey` bytes, enabling backfill of the `agent_key_ref` column without re-signing.

This aligns with Sigstore/Rekor's key identity model (Rekor v2 GA, 2025) and NIST SP 800-57 Compromised Key List semantics.

---

### `SigningKey` record

```java
package io.casehub.ledger.runtime.service;

public record SigningKey(String keyRef, KeyPair keyPair) {

    public static SigningKey of(KeyPair keyPair) {
        byte[] encoded = keyPair.getPublic().getEncoded();
        byte[] hash = MessageDigest.getInstance("SHA-256").digest(encoded);
        String keyRef = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
        return new SigningKey(keyRef, keyPair);
    }
}
```

---

### `AgentKeyProvider` SPI change

```java
// Before (breaking — return type changes)
Optional<KeyPair> signingKeyPair(String actorId);

// After
Optional<SigningKey> signingKey(String actorId);
```

Only `ConfiguredAgentKeyProvider` implements this in the ledger repo — no downstream implementations exist.

---

### Schema — V1006 and V1007

**V1006** — add `agent_key_ref` to `ledger_entry`:
```sql
ALTER TABLE ledger_entry ADD COLUMN agent_key_ref TEXT;
ALTER TABLE ledger_entry ADD CONSTRAINT chk_agent_key_ref_pair
    CHECK ((agent_key_ref IS NULL) = (agent_signature IS NULL));
```

**V1007** — new `key_rotation_entry` table (JOINED inheritance subclass):
```sql
CREATE TABLE key_rotation_entry (
    id              UUID PRIMARY KEY REFERENCES ledger_entry(id),
    previous_key_ref TEXT,
    new_key_ref      TEXT,
    reason           TEXT NOT NULL,
    effective_since  TIMESTAMP WITH TIME ZONE NOT NULL
);
```

`reason` stores `SCHEDULED` or `COMPROMISED`. `new_key_ref` is nullable — a pure revocation without replacement leaves it null.

---

### `KeyRotationReason` enum

```java
package io.casehub.ledger.api.model;

public enum KeyRotationReason {
    SCHEDULED,    // planned cryptoperiod rotation
    COMPROMISED   // key was leaked or actor was misbehaving
}
```

Lives in the `api` module — consumers need it to record rotations.

---

### `KeyRotationEntry`

`LedgerEntry` subclass. `subjectId` is `UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` — deterministic, actor-scoped. Makes the key lifecycle queryable as a ledger sequence per actor. `entryType = COMMAND` (a deliberate operator action).

```java
@Entity
@Table(name = "key_rotation_entry")
@DiscriminatorValue("KEY_ROTATION")
public class KeyRotationEntry extends LedgerEntry {
    @Column(name = "previous_key_ref")
    public String previousKeyRef;       // null when the previous key is unknown

    @Column(name = "new_key_ref")
    public String newKeyRef;            // null for pure revocation without replacement

    @Enumerated(EnumType.STRING)
    @Column(name = "reason", nullable = false)
    public KeyRotationReason reason;

    @Column(name = "effective_since", nullable = false)
    public Instant effectiveSince;      // for COMPROMISED: entries signed after this are suspect
}
```

---

### `KeyRotationRepository`

SPI interface:

```java
public interface KeyRotationRepository {
    KeyRotationEntry save(KeyRotationEntry entry);
    List<KeyRotationEntry> findByActorId(String actorId);
    List<KeyRotationEntry> findCompromisedByActorIdAndKeyRef(String actorId, String keyRef);
}
```

JPA implementation: `JpaKeyRotationRepository` using `@NamedQuery` on `KeyRotationEntry`.

**Note:** Adding `KeyRotationRepository` as a new SPI does NOT trigger `ledger-spi-propagation` — that protocol applies only to `LedgerEntryRepository` and `ReactiveLedgerEntryRepository`. `KeyRotationRepository` is a separate SPI.

---

### `KeyRotationService`

CDI bean:

```java
@ApplicationScoped
public class KeyRotationService {

    /** Record a key rotation event. newKeyRef is null for pure revocations. */
    @Transactional
    public KeyRotationEntry recordRotation(
            String actorId,
            String previousKeyRef,
            @Nullable String newKeyRef,
            KeyRotationReason reason,
            Instant effectiveSince);

    /** All rotation events for an actor, ordered by occurredAt. */
    @Transactional
    public List<KeyRotationEntry> rotationHistory(String actorId);

    /**
     * Returns all time windows during which a specific key was reported compromised
     * for a given actor. Used by LedgerVerificationService to detect SUSPECT entries.
     */
    @Transactional
    public List<CompromisedWindow> compromisedWindows(String actorId, String keyRef);
}
```

`CompromisedWindow` — value type: `record CompromisedWindow(String keyRef, Instant effectiveSince)`.

---

### `AgentSignatureEnricher` update

Stores `entry.agentKeyRef = signingKey.keyRef()` alongside the existing `agentSignature` and `agentPublicKey` fields.

---

### `VerificationResult` — add `SUSPECT`

```java
public enum VerificationResult {
    UNSIGNED,   // no signature stored
    VALID,      // signature present and cryptographically verified
    INVALID,    // signature present but verification failed — tamper detected
    SUSPECT     // signature is VALID but signed under a key subsequently reported COMPROMISED
}
```

---

### `LedgerVerificationService.verifyAgentSignature` update

After the existing VALID check, query `KeyRotationService.compromisedWindows(entry.actorId, entry.agentKeyRef)`. If any window's `effectiveSince` is before `entry.occurredAt`, return `SUSPECT`.

```
UNSIGNED → no agentSignature stored
INVALID  → cryptographic failure
VALID    → signature checks out, key not in any compromise window
SUSPECT  → signature checks out, but key was COMPROMISED and entry.occurredAt >= effectiveSince
```

---

### `ConfiguredAgentKeyProvider` update

`loadKeys()` now wraps each loaded `KeyPair` in `SigningKey.of(keyPair)`. The `keyRef` is derived at load time and logged alongside the actor confirmation.

---

## What's out of scope

| Concern | Tracked |
|---------|---------|
| CDI event on SUSPECT detection | #83 |
| Post-quantum algorithm migration | #84 |
| External key distribution (TUF/HSM/PKI) | #85 |
| Backfilling `agent_key_ref` on existing entries | Not needed — column nullable, old entries remain verifiable |

---

## Testing strategy

**Unit — `SigningKeyTest`**
- `SigningKey.of()` derives a consistent keyRef from the same public key
- Two different public keys produce different keyRefs

**Unit — `KeyRotationServiceTest`**
- `recordRotation` creates a `KeyRotationEntry` with correct `subjectId` derivation
- `compromisedWindows` returns correct windows for COMPROMISED records only

**Unit — `AgentSignatureEnricherTest` update**
- Enrich now populates `agentKeyRef` matching `SigningKey.keyRef()`

**Unit — `LedgerVerificationServiceTest` update**
- New case: SUSPECT when entry's keyRef appears in a compromised window before `occurredAt`
- VALID still returned when compromise window is after `occurredAt`
- VALID returned when no compromise record exists

**Integration — `KeyRotationIT` (QuarkusTest)**
- End-to-end: persist signed entry → record COMPROMISED rotation → `verifyAgentSignature` returns SUSPECT
- SCHEDULED rotation → VALID entries before rotation remain VALID; VALID entries after use new keyRef
- `rotationHistory(actorId)` returns entries in sequence

**ADR 0012** — key versioning and rotation design: self-derived keyRef rationale, NIST SP 800-57 / Sigstore alignment, SCHEDULED vs COMPROMISED distinction.
