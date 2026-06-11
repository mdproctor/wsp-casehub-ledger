# Merkle Content Integrity — Hash the Letter, Not Just the Envelope

**Issue:** casehubio/ledger#128
**Date:** 2026-06-11
**Status:** Design

---

## Problem

The Merkle leaf hash covers 6 structural fields:

```
subjectId|seqNum|entryType|actorId|actorRole|occurredAt
```

This proves "entry N happened, by actor Y, at time T." It does not prove what
the entry contained. A privileged insider with database access can change
`caseStatus = REJECTED` to `APPROVED`, `confidenceScore = 0.3` to `0.95`, or
`reason = COMPROMISED` to `SCHEDULED` — without breaking the chain or
invalidating the agent signature.

The agent signature shares `canonicalBytes()` with the Merkle hash — it covers
the same 6 fields. The bilateral signing infrastructure proves the agent signed
an entry, but not what the entry said.

Application-tier consumers (devtown, aml, clinical) cite the Merkle chain as
evidence of tamper-evidence for compliance reports. The chain does not deliver
what they think it delivers.

## Governing Principle

RFC 9162 (Certificate Transparency) hashes the full certificate. Git hashes the
full tree. Blockchain hashes all transactions. No mainstream append-only
authenticated log deliberately excludes content from its leaf hash. The current
design is the outlier.

A tamper-evident audit ledger that does not protect content is not tamper-evident
in any meaningful sense. The existing DESIGN.md rationale — "domain labels do
not participate in tamper detection" — treated implementation convenience as a
security principle. This spec corrects that.

---

## 1. Canonical Form — What the Hash Covers

### Current (6 fields)

```
subjectId|seqNum|entryType|actorId|actorRole|occurredAt
```

### After (structural + content)

```
subjectId|seqNum|entryType|actorId|actorRole|occurredAt|tenancyId|actorType|causedByEntryId|supplementJson|domainContent
```

**Added base-class fields:**

| Field | Why |
|-------|-----|
| `tenancyId` | Tampering moves entry to wrong tenant |
| `actorType` | Changing HUMAN↔AGENT alters semantic meaning |
| `causedByEntryId` | Tampering falsifies the causal chain |

**Added content:**

| Content | Source | Why |
|---------|--------|-----|
| `supplementJson` | Base table column | Compliance/provenance metadata — the payload that GDPR/EU AI Act compliance reports cite |
| Domain content | `domainContentBytes()` override | Subclass join-table fields — `caseStatus`, `messageType`, `content`, `previousKeyRef`, etc. |

**Excluded (with rationale):**

| Field | Why excluded |
|-------|-------------|
| `traceId` | OTel correlation — informational, not semantic |
| `actorDid` | DID binding has its own verification mechanism (DID resolution + key match) |
| `agentSignature` / `agentPublicKey` / `agentKeyRef` | The signature can't be inside its own hash (circular) |
| `digest` | The hash can't be inside itself |
| `id` | Database-assigned PK — not a content field |

---

## 2. `canonicalBytes()` Moves to `LedgerEntry`

`canonicalBytes()` is currently a `public static` method on `LedgerMerkleTree`.
With content hashing, it calls `entry.domainContentBytes()` — a virtual dispatch.
A static utility calling an instance method on its parameter is semantically wrong.
The canonical form is a property of the entry, not of the Merkle tree utility.

**Move to `LedgerEntry` as a public instance method:**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java
public byte[] canonicalBytes() {
    final String structural = String.join("|",
        subjectId != null ? subjectId.toString() : "",
        String.valueOf(sequenceNumber),
        entryType != null ? entryType.name() : "",
        actorId != null ? actorId : "",
        actorRole != null ? actorRole : "",
        occurredAt != null ? occurredAt.truncatedTo(ChronoUnit.MILLIS).toString() : "",
        tenancyId != null ? tenancyId : "",
        actorType != null ? actorType.name() : "",
        causedByEntryId != null ? causedByEntryId.toString() : "");

    final String withSupplements = supplementJson != null
        ? structural + "|" + supplementJson
        : structural;

    final byte[] base = withSupplements.getBytes(StandardCharsets.UTF_8);
    final byte[] domain = domainContentBytes();

    if (domain.length == 0) {
        return base;
    }

    final byte[] combined = new byte[base.length + 1 + domain.length];
    System.arraycopy(base, 0, combined, 0, base.length);
    combined[base.length] = (byte) '|';
    System.arraycopy(domain, 0, combined, base.length + 1, domain.length);
    return combined;
}
```

**`LedgerMerkleTree.canonicalBytes(LedgerEntry)` becomes a delegation:**

```java
public static byte[] canonicalBytes(final LedgerEntry entry) {
    return entry.canonicalBytes();
}
```

Mark `@Deprecated(forRemoval = true)` — callers should migrate to
`entry.canonicalBytes()`. The static method stays temporarily to avoid a
flag-day migration across all repos.

**Call site migration (13 sites in ledger):**

| Caller | Change |
|--------|--------|
| `LedgerMerkleTree.leafHash()` | `canonicalBytes(entry)` → `entry.canonicalBytes()` |
| `AgentSignatureEnricher.enrich()` | `LedgerMerkleTree.canonicalBytes(entry)` → `entry.canonicalBytes()` |
| `AgentCryptographicVerifier.verifyCryptographic()` | Same |
| Test classes (8 sites) | Same |

Downstream repos (`engine`, `qhorus`) use `LedgerMerkleTree.canonicalBytes()` in
test helpers — the deprecated static delegation gives them time to migrate.

---

## 3. `domainContentBytes()` — Subclass Extension Point

```java
// runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java

/**
 * Returns domain-specific content bytes for hash protection.
 *
 * <p>Subclasses that declare {@code @Column} fields on join tables MUST override
 * this method to include those fields. The returned bytes are appended to the
 * canonical form used by both the Merkle leaf hash and the agent signature.
 *
 * <p>Build-time enforcement: {@code LedgerProcessor} produces a deployment error
 * if a {@code LedgerEntry} subclass declares {@code @Column} fields but does not
 * override this method.
 *
 * @return domain content bytes; empty array if no domain fields exist
 */
protected byte[] domainContentBytes() {
    return EMPTY_BYTES;
}

private static final byte[] EMPTY_BYTES = new byte[0];
```

**Why not abstract:** `LedgerEntry` has base-table content (`supplementJson`)
that it handles itself in `canonicalBytes()`. An abstract `domainContentBytes()`
would say "I don't know my content" — but the base class does know its supplement
content. The override point is specifically for domain content on join tables,
which the base class correctly knows nothing about.

### Internal subclass implementations

**`PlainLedgerEntry`** — no domain fields. Default empty bytes. No override needed.

**`KeyRotationEntry`:**

```java
@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        previousKeyRef != null ? previousKeyRef : "",
        newKeyRef != null ? newKeyRef : "",
        reason != null ? reason.name() : "",
        effectiveSince != null ? effectiveSince.truncatedTo(ChronoUnit.MILLIS).toString() : ""
    ).getBytes(StandardCharsets.UTF_8);
}
```

**`ActorIdentityBindingEntry`:**

```java
@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        boundDid != null ? boundDid : "",
        validationResult != null ? validationResult.name() : "",
        String.valueOf(alsoKnownAsVerified),
        String.valueOf(keyMatchVerified),
        verifiedKeyRef != null ? verifiedKeyRef : "",
        credentialResult != null ? credentialResult.name() : "",
        didMethod != null ? didMethod : ""
    ).getBytes(StandardCharsets.UTF_8);
}
```

### Downstream subclasses (separate issues)

| Subclass | Repo | Issue | Fields to include |
|----------|------|-------|-------------------|
| `CaseLedgerEntry` | engine | new issue | caseId, commandType, eventType, caseStatus |
| `MessageLedgerEntry` | qhorus | new issue | channelId, messageId, messageType, target, content, correlationId, commitmentId, toolName, durationMs, tokenCount, contextRefs, sourceEntity |

---

## 4. Build-Time Guard — `domainContentBytes` Override Enforcement

Add a `@BuildStep` in `LedgerProcessor` that scans Jandex for `LedgerEntry`
subclasses declaring `@Column` fields without overriding `domainContentBytes()`.

```java
@BuildStep
void validateDomainContentBytes(CombinedIndexBuildItem index) {
    // For each LedgerEntry subclass in Jandex:
    //   1. Collect @Column fields declared on the subclass (not inherited)
    //   2. Check if domainContentBytes() is overridden
    //   3. If columns exist but no override → deployment error
    //
    // PlainLedgerEntry has zero @Column fields → no override needed → passes
    // KeyRotationEntry has @Column fields + override → passes
    // CaseLedgerEntry has @Column fields + no override → ERROR
}
```

**Deployment error message:**

> `CaseLedgerEntry declares @Column fields (caseId, commandType, eventType, caseStatus)
> but does not override domainContentBytes(). These fields are not hash-protected.
> Override domainContentBytes() to include them in the Merkle leaf hash and agent signature.`

This catches downstream consumers at build time — the same enforcement pattern as
the field-shadowing guard from #131.

---

## 5. Save Pipeline Restructure — Enrich → Hash → Sign → Persist

### Current JPA pipeline (broken ordering)

```
save():
  1. stamp tenancyId, occurredAt, seqNum
  2. digest = leafHash(entry)           ← BEFORE enrichers
  3. em.persist()
       └→ @PrePersist → LedgerTraceListener → enricherPipeline.enrich()
            └→ TraceIdEnricher        (10)  — sets traceId
            └→ AgentSignatureEnricher (20)  — signs canonicalBytes()
            └→ ProvenanceCaptureEnricher (30) — attaches ProvenanceSupplement
            └→ ActorDIDEnricher       (40)  — sets actorDid
            └→ ActorIdentityValidation(50)  — validates DID/VC
  4. Merkle frontier update
```

The digest is computed at step 2 before enrichers at step 3. The signature at
priority 20 runs before provenance at priority 30. Neither the digest nor the
signature can cover enricher-set data.

The in-memory save already has the right ordering: enrichers → digest → store.

### New pipeline (both paths)

```
save():
  1. stamp tenancyId, occurredAt, seqNum
  2. enricherPipeline.enrichContent(entry)     ← content enrichers only
       └→ TraceIdEnricher        (10)
       └→ ProvenanceCaptureEnricher (30)
       └→ ActorDIDEnricher       (40)
       └→ ActorIdentityValidation(50)
  3. digest = leafHash(entry)                   ← AFTER content enrichment
  4. enricherPipeline.seal(entry)               ← sealing only
       └→ AgentSignatureEnricher (20) → signs entry.canonicalBytes()
  5. em.persist()                                ← no enricher pipeline
  6. Merkle frontier update
```

### Implementation changes

**`LedgerEntryEnricher` gains a phase marker:**

```java
public interface LedgerEntryEnricher {
    void enrich(LedgerEntry entry);

    default Phase phase() {
        return Phase.CONTENT;
    }

    enum Phase { CONTENT, SEAL }
}
```

`AgentSignatureEnricher` overrides: `phase() → Phase.SEAL`.
All other enrichers keep the default `Phase.CONTENT`.

**`LedgerEnricherPipeline` gains phase-aware methods:**

```java
public void enrichContent(final LedgerEntry entry) {
    run(entry, Phase.CONTENT);
}

public void seal(final LedgerEntry entry) {
    run(entry, Phase.SEAL);
}

// existing enrich() method deprecated — calls both phases for backward compat
@Deprecated(forRemoval = true)
public void enrich(final LedgerEntry entry) {
    run(entry, Phase.CONTENT);
    run(entry, Phase.SEAL);
}

private void run(final LedgerEntry entry, final Phase phase) {
    enrichers.handlesStream()
        .sorted(Comparator.comparingInt(h ->
            (h.getBean() instanceof InjectableBean<?> ib) ? ib.getPriority() : Integer.MAX_VALUE))
        .map(h -> h.get())
        .filter(e -> e.phase() == phase)
        .forEach(e -> { /* existing try/catch */ });
}
```

**`LedgerTraceListener` — remove enricher pipeline invocation:**

The listener currently runs the enricher pipeline at `@PrePersist`. With the
pipeline moved to explicit save() steps, the listener must not re-run it.

Replace with a defensive check:

```java
@PrePersist
public void prePersist(final Object entity) {
    if (!(entity instanceof LedgerEntry entry)) return;
    if (ledgerConfig.hashChain().enabled() && entry.digest == null) {
        throw new IllegalStateException(
            "LedgerEntry persisted without digest — use LedgerEntryRepository.save(), "
            + "not em.persist() directly");
    }
}
```

This catches direct `em.persist()` calls that bypass the repository contract.

**`JpaLedgerEntryRepository.save()` — new ordering:**

```java
public LedgerEntry save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId;
    // ... existing validation, tokenisation, sanitisation, seqNum allocation ...

    enricherPipeline.enrichContent(entry);          // Phase 1: content enrichment

    if (ledgerConfig.hashChain().enabled()) {
        entry.digest = LedgerMerkleTree.leafHash(entry);  // Phase 2: hash
    }

    enricherPipeline.seal(entry);                   // Phase 3: seal (agent signature)

    em.persist(entry);                               // Phase 4: persist (no enrichers)

    if (ledgerConfig.hashChain().enabled()) {
        updateMerkleFrontier(entry, tenancyId);
    }
    return entry;
}
```

**`InMemoryLedgerEntryRepository.save()` — minor restructure:**

The in-memory path already runs enrichers before digest. Change: split the
`enricherPipeline.enrich(entry)` call into `enrichContent()` + digest + `seal()`.

---

## 6. `supplementJson` Serialization Stability

`LedgerSupplementSerializer.toJson()` uses `LinkedHashMap` with explicit field
ordering in `toFieldMap()`. Jackson serializes in insertion order. The output
is deterministic for the same logical content — no serialization instability
concern.

---

## 7. Verification Impact

**Merkle verification** (`LedgerVerificationService.verify()`): loads the full
entry via `findEntryById()`. JPA JOINED inheritance returns the concrete subclass.
`leafHash(entry)` calls `entry.canonicalBytes()` which dispatches to the right
`domainContentBytes()` override. No API changes needed.

**Agent signature verification** (`AgentCryptographicVerifier.verifyCryptographic()`):
same — receives the full entry, calls `entry.canonicalBytes()`. Signature coverage
expands to match the digest. No API changes.

**Inclusion proof verification** (`LedgerMerkleTree.verifyProof()`): operates on
hex strings — no change.

---

## 8. What Stays the Same

- `LedgerMerkleTree` algorithms (append, treeRoot, inclusionProof, verifyProof)
- `LedgerVerificationService` API
- `AgentSigner` SPI
- Supplement model and `attach()`
- No existing installations — no data migration
- All Merkle frontiers can be rebuilt from scratch (test infrastructure)

---

## 9. Downstream Propagation

Breaking change: `LedgerEntry` subclasses with `@Column` fields that do not
override `domainContentBytes()` will fail at build time.

| Repo | Subclass | Status |
|------|----------|--------|
| engine | `CaseLedgerEntry` | engine#471 — add `domainContentBytes()` override |
| qhorus | `MessageLedgerEntry` | qhorus#270 — add `domainContentBytes()` override |

These are mechanical — each override is a single method returning `String.join("|", field1, field2, ...).getBytes(UTF_8)`.

---

## 10. Test Strategy

**`LedgerMerkleTree` unit tests:**
- Existing `canonicalBytes` tests updated: assert new fields are present in output
- New: verify `domainContentBytes()` contribution appears in leaf hash
- New: verify `PlainLedgerEntry` (empty domain) produces same structural hash as before + new base fields

**`KeyRotationEntry` / `ActorIdentityBindingEntry` content integrity tests:**
- Mutate a domain field after digest computation → `leafHash(entry) != entry.digest`
- Verify `domainContentBytes()` includes all `@Column` fields

**Pipeline ordering tests:**
- Verify `ProvenanceSupplement` attached by enricher IS reflected in digest (was NOT before)
- Verify `AgentSignatureEnricher` runs AFTER digest computation
- Verify signature covers the same canonical bytes as the digest

**`LedgerTraceListener` defensive check:**
- Direct `em.persist()` without digest → `IllegalStateException`

**Build-time guard test:**
- Subclass with `@Column` fields and no `domainContentBytes()` override → deployment error

**Existing test updates:**
- All tests that construct `LedgerEntry` instances and compute/compare digests need
  updating for the expanded canonical form (more fields in the hash)
- `AgentCryptographicVerifierTest` — canonical bytes include new fields
- `AgentSignatureEnricherTest` — signature covers expanded canonical bytes
- `LedgerMerkleTreeTest` — leaf hash values change

---

## 11. DESIGN.md Updates

Replace the "Hash chain canonical form" section:

**Before:**
> `subjectId|seqNum|entryType|actorId|actorRole|occurredAt`
> Supplement fields are deliberately excluded from the chain: the chain covers the
> immutable core audit record; compliance metadata is enrichment, not a tamper-evidence target.

**After:**
> The leaf hash covers all tamper-critical fields: structural metadata, base-table
> supplements, and subclass domain content via `domainContentBytes()`.
> `canonicalBytes()` is an instance method on `LedgerEntry` — the canonical form
> is a property of the entry, not of the Merkle tree utility.
>
> The enricher pipeline runs in two phases: content enrichment (provenance, DID,
> traceId) → hashing → sealing (agent signature). The signature covers exactly
> what the digest covers.
