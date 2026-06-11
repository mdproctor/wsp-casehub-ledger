# Merkle Content Integrity — Hash the Letter, Not Just the Envelope

**Issue:** casehubio/ledger#128
**Date:** 2026-06-11
**Status:** Design — revision 3

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

## ARC42 Dependency

ARC42STORIES line 149: "Ch2 before Ch3: supplement schema (V1002) must settle
before Merkle canonical form is finalised." V1002 has been stable since April.
This change finalises the canonical form to include supplement content —
completing the Ch2→Ch3 dependency.

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

**Move to `runtime.model.LedgerEntry` as a public final instance method.**

The `api.model.LedgerEntry` does NOT get this method — it lacks agent signing fields
(`agentSignature`, `agentPublicKey`, `agentKeyRef`) and `actorDid`, so it cannot
compute the full canonical form. The api module is model-only; hashing is a runtime
concern. `domainContentBytes()` goes on the runtime class only, since downstream
subclasses (`CaseLedgerEntry`, `MessageLedgerEntry`) extend the runtime class.

```java
// runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java
public final byte[] canonicalBytes() {
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

**Delete `LedgerMerkleTree.canonicalBytes(LedgerEntry)` entirely.** No deprecated
delegation. This platform has no end users — breaking changes cost nothing
externally. Downstream repos get a clean compile error pointing them to
`entry.canonicalBytes()`. The migration is one-line mechanical.

**Call site migration (13 sites in ledger):**

| Caller | Change |
|--------|--------|
| `LedgerMerkleTree.leafHash()` | `canonicalBytes(entry)` → `entry.canonicalBytes()` |
| `AgentEntrySigner.sign()` (was AgentSignatureEnricher) | `LedgerMerkleTree.canonicalBytes(entry)` → `entry.canonicalBytes()` |
| `AgentCryptographicVerifier.verifyCryptographic()` | Same |
| Test classes (8 sites) | Same |

---

## 3. `domainContentBytes()` — Subclass Extension Point

```java
// runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java

/**
 * Returns domain-specific content bytes for hash protection.
 *
 * <p>Subclasses that declare persistent fields on join tables MUST override
 * this method to include those fields. The returned bytes are appended to the
 * canonical form used by both the Merkle leaf hash and the agent signature.
 *
 * <p>Build-time enforcement: {@code LedgerProcessor} produces a deployment error
 * if a {@code LedgerEntry} subclass declares persistent fields (non-{@code @Transient})
 * but does not override this method.
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
which the base class correctly knows nothing about. `PlainLedgerEntry` has no
domain fields — an empty default is correct, not ceremony.

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
| `CaseLedgerEntry` | engine | engine#471 | caseId, commandType, eventType, caseStatus |
| `MessageLedgerEntry` | qhorus | qhorus#270 | channelId, messageId, messageType, target, content, correlationId, commitmentId, toolName, durationMs, tokenCount, contextRefs, sourceEntity |

---

## 4. Build-Time Guard — `domainContentBytes` Override Enforcement

Add a `@BuildStep` in `LedgerProcessor` that scans Jandex for `LedgerEntry`
subclasses declaring persistent fields without overriding `domainContentBytes()`.

**Filtering:**
- Only `@Entity`-annotated subclasses are scanned. Test subclasses without `@Entity`
  (15 in ledger: `MemoryTestEntry`, `TestLedgerEntry`, `ConcreteEntry`, etc.) are
  excluded — they are plain Java helpers, not JPA entities.
- A field is persistent if it is NOT annotated `@Transient`. JPA persists all
  non-transient fields by default on `@Entity` classes — checking for `@Column`
  would miss fields that rely on JPA's default mapping (e.g., a field with only
  `@Enumerated`).

```java
@BuildStep
void validateDomainContentBytes(CombinedIndexBuildItem index) {
    final DotName ENTITY = DotName.createSimple("jakarta.persistence.Entity");
    final DotName TRANSIENT = DotName.createSimple("jakarta.persistence.Transient");
    final DotName LEDGER_ENTRY = DotName.createSimple(LedgerEntry.class.getName());

    for (ClassInfo subclass : index.getIndex().getAllKnownSubclasses(LEDGER_ENTRY)) {
        if (!subclass.hasAnnotation(ENTITY)) continue;  // skip test helpers

        // Check for non-@Transient declared fields
        boolean hasPersistentFields = subclass.fields().stream()
            .anyMatch(f -> !f.hasAnnotation(TRANSIENT));

        if (!hasPersistentFields) continue;  // PlainLedgerEntry, TestEntry — no domain fields

        // Check if domainContentBytes() is overridden
        boolean overrides = subclass.method("domainContentBytes") != null;

        if (!overrides) {
            String fieldNames = subclass.fields().stream()
                .filter(f -> !f.hasAnnotation(TRANSIENT))
                .map(FieldInfo::name)
                .collect(Collectors.joining(", "));
            throw new DeploymentException(
                subclass.name().withoutPackagePrefix()
                + " declares persistent fields (" + fieldNames
                + ") but does not override domainContentBytes(). "
                + "These fields are not hash-protected. Override domainContentBytes() "
                + "to include them in the Merkle leaf hash and agent signature.");
        }
    }
}
```

---

## 5. Save Pipeline Restructure — Enrich → Hash → Sign → Persist

### Current JPA pipeline (broken ordering)

```
save():
  1. stamp tenancyId, occurredAt, seqNum
  2. digest = leafHash(entry)             ← BEFORE enrichers
  3. em.persist()
       └→ @PrePersist → LedgerTraceListener → enricherPipeline.enrich()
            └→ AgentSignatureEnricher  (20) — signs canonicalBytes()
            └→ ActorDIDEnricher        (40) — sets actorDid
            └→ ActorIdentityValidation (50) — validates DID/VC
            └→ TraceIdEnricher     (MAX_VALUE) — sets traceId (NO @Priority!)
            └→ ProvenanceCaptureEnricher (MAX_VALUE) — attaches ProvenanceSupplement (NO @Priority!)
```

**Missing `@Priority` annotations:** `TraceIdEnricher` and `ProvenanceCaptureEnricher`
have no `@Priority` annotation despite the `LedgerEntryEnricher` javadoc claiming
priorities 10 and 30. They sort to `Integer.MAX_VALUE` and run in undefined order
relative to each other. This means the current execution order is: signing (20),
then DID (40), then identity validation (50), then traceId and provenance in
arbitrary order. The spec's pipeline diagram in revision 1 was wrong.

**Fix:** Add `@Priority(10)` to `TraceIdEnricher` and `@Priority(30)` to
`ProvenanceCaptureEnricher` as part of this change. The in-memory path's comment
at line 108-111 assumed enricher ordering was correct — it wasn't.

### Signing is not enrichment — extract from pipeline

`AgentSignatureEnricher` is the only enricher with a hard ordering dependency on
the digest. It doesn't ADD content to the entry — it SEALS it. Treating it as
"enricher at priority 20" conflates two different concerns and forced the Phase
abstraction in revision 1, which added framework-level complexity for a single case.

**Extract signing from the enricher pipeline entirely.** Create `AgentEntrySigner`
— a thin CDI bean that both repositories inject directly:

```java
@ApplicationScoped
public class AgentEntrySigner {

    private static final Logger LOG = Logger.getLogger(AgentEntrySigner.class);

    private final AgentSigner signer;

    @Inject
    public AgentEntrySigner(final AgentSigner signer) {
        this.signer = signer;
    }

    public void sign(final LedgerEntry entry) {
        if (entry.actorId == null || entry.agentSignature != null) return;
        try {
            signer.sign(entry.actorId, entry.canonicalBytes())
                    .ifPresent(sig -> {
                        entry.agentSignature = sig.signature();
                        entry.agentPublicKey = sig.publicKey();
                        entry.agentKeyRef = sig.keyRef();
                    });
        } catch (final Exception e) {
            LOG.warnf("AgentEntrySigner failed for actor %s — entry will be unsigned: %s",
                    entry.actorId, e.getMessage());
        }
    }
}
```

Delete `AgentSignatureEnricher`. No `Phase` enum, no `enrichContent()`/`seal()`
split, no deprecated `enrich()` method. `LedgerEntryEnricher` stays a
single-concern interface — all enrichers are content enrichers.

### New pipeline (both paths)

```
save():
  1. stamp tenancyId, occurredAt, seqNum
  2. enricherPipeline.enrich(entry)         ← content enrichers only
       └→ TraceIdEnricher              (10)
       └→ ProvenanceCaptureEnricher    (30)
       └→ ActorDIDEnricher             (40)
       └→ ActorIdentityValidation      (50)
  3. digest = leafHash(entry)               ← AFTER content enrichment
  4. agentEntrySigner.sign(entry)           ← signs entry.canonicalBytes()
  5. em.persist()                            ← no enricher pipeline
  6. Merkle frontier update
```

### `LedgerTraceListener` — defensive check

Replace the enricher pipeline invocation with a guard that catches direct
`em.persist()` calls bypassing the repository:

```java
@PrePersist
public void prePersist(final Object entity) {
    if (!(entity instanceof LedgerEntry entry)) return;
    if (ledgerConfig.hashChain().enabled() && entry.digest == null) {
        throw new IllegalStateException(
            "LedgerEntry must be persisted through LedgerEntryRepository, "
            + "which handles sequence allocation, enrichment, hashing, and signing. "
            + "Direct em.persist() bypasses the entire save pipeline.");
    }
}
```

### `LedgerEntryEnricher` javadoc — full rewrite

The current contract says enrichers must not modify "canonical fields" (the old 6).
After this change, the entire rationale flips: enrichers run BEFORE hashing, and
the canonical form expands. The enricher contract must be rewritten:

```java
/**
 * SPI for auto-populating fields on {@link LedgerEntry} at persist time.
 *
 * <p><strong>Pipeline position:</strong> Content enrichers run BEFORE hashing
 * and signing. The full save pipeline is:
 * enrichment → digest (leafHash) → agent signature → persist.
 *
 * <p><strong>Ordering:</strong> Enrichers are invoked in ascending
 * {@code @Priority} order. All implementations MUST carry a
 * {@code @jakarta.annotation.Priority} annotation:
 * <ul>
 *   <li>10 — TraceIdEnricher</li>
 *   <li>30 — ProvenanceCaptureEnricher</li>
 *   <li>40 — ActorDIDEnricher</li>
 *   <li>50 — ActorIdentityValidationEnricher</li>
 * </ul>
 *
 * <p><strong>Contract:</strong>
 * <ul>
 *   <li>Do NOT overwrite fields stamped by the save pipeline:
 *       {@code subjectId}, {@code sequenceNumber}, {@code tenancyId},
 *       {@code occurredAt}.</li>
 *   <li>Enrichers MAY attach supplements — that is the point of
 *       {@link io.casehub.ledger.runtime.service.intercept.ProvenanceCaptureEnricher}.</li>
 *   <li>Enrichers that modify supplement fields in-place MUST call
 *       {@link LedgerEntry#refreshSupplementJson()} or
 *       {@link LedgerEntry#attach(LedgerSupplement)} — direct field mutation
 *       leaves {@code supplementJson} stale, and the hash will cover the
 *       stale value.</li>
 * </ul>
 */
```

### `JpaLedgerEntryRepository.save()` — new ordering

```java
public LedgerEntry save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId;
    // ... existing validation, tokenisation, sanitisation, seqNum allocation ...

    enricherPipeline.enrich(entry);                       // content enrichment

    if (ledgerConfig.hashChain().enabled()) {
        entry.digest = LedgerMerkleTree.leafHash(entry);  // hash
    }

    agentEntrySigner.sign(entry);                         // seal

    em.persist(entry);                                     // persist (no enrichers)

    if (ledgerConfig.hashChain().enabled()) {
        updateMerkleFrontier(entry, tenancyId);
    }
    return entry;
}
```

**`InMemoryLedgerEntryRepository.save()`** — same restructure: replace
`enricherPipeline.enrich(entry)` with the three-phase sequence. The in-memory
path already ran enrichers before digest; this change formalises signing as a
separate step.

---

## 6. `supplementJson` Serialization Stability

`LedgerSupplementSerializer.toJson()` uses `LinkedHashMap` with explicit field
ordering in `toFieldMap()`. Jackson serializes in insertion order. The output
is deterministic for the same logical content.

**Precision:** Determinism is path-dependent — it depends on supplement attachment
order. If `COMPLIANCE` is attached before `PROVENANCE`, the JSON key order is
`{"COMPLIANCE":{...},"PROVENANCE":{...}}`; reverse the attachment order, and the
JSON changes. In practice, attachment order is deterministic within a given code
path (callers attach supplements in a fixed sequence). Verification uses the
**stored** `supplementJson` column value, not a re-serialization — so even if a
future code change reorders attachments, existing entries remain verifiable.

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

Breaking change: `LedgerEntry` subclasses with persistent fields that do not
override `domainContentBytes()` will fail at build time.
`LedgerMerkleTree.canonicalBytes(LedgerEntry)` is deleted — callers get a
compile error and migrate to `entry.canonicalBytes()`.

| Repo | Subclass | Issue |
|------|----------|-------|
| engine | `CaseLedgerEntry` | engine#471 — add `domainContentBytes()` override |
| qhorus | `MessageLedgerEntry` | qhorus#270 — add `domainContentBytes()` override |

---

## 10. Test Strategy

**`canonicalBytes()` unit tests (on `LedgerEntry`):**
- Verify new base fields (tenancyId, actorType, causedByEntryId) are present in output
- Verify supplementJson appears when supplements are attached
- Verify `domainContentBytes()` contribution appears in the canonical bytes
- Verify `PlainLedgerEntry` hash includes new base fields and produces no domain
  content contribution (empty `domainContentBytes()`)

**`KeyRotationEntry` / `ActorIdentityBindingEntry` content integrity tests:**
- Mutate a domain field after digest computation → `leafHash(entry) != entry.digest`
- Verify `domainContentBytes()` includes all persistent fields

**Pipeline ordering tests:**
- Verify `ProvenanceSupplement` attached by enricher IS reflected in digest
  (was NOT before — enricher ran after hashing in JPA path)
- Verify `AgentEntrySigner.sign()` runs AFTER digest computation
- Verify signature covers the same canonical bytes as the digest

**`LedgerTraceListener` defensive check:**
- Direct `em.persist()` without digest → `IllegalStateException` with message
  naming the full pipeline (sequence allocation, enrichment, hashing, signing)

**Build-time guard test:**
- `@Entity` subclass with persistent fields and no `domainContentBytes()` override
  → deployment error
- `@Entity` subclass with no persistent fields (e.g. `PlainLedgerEntry`) → passes
- Non-`@Entity` test subclass with fields → not scanned, passes

**Existing test updates:**
- All tests that compute/compare digests need updating for the expanded canonical form
- `AgentCryptographicVerifierTest` — canonical bytes include new fields
- Test helpers calling `LedgerMerkleTree.canonicalBytes(entry)` → `entry.canonicalBytes()`

---

## 11. DESIGN.md Updates

Replace the "Hash chain canonical form" section:

**Before:**
> `subjectId|seqNum|entryType|actorId|actorRole|occurredAt`
> Supplement fields are deliberately excluded from the chain: the chain covers the
> immutable core audit record; compliance metadata is enrichment, not a tamper-evidence target.
> Deliberately excludes subclass-specific fields. The chain covers provenance and timing;
> domain labels do not participate in tamper detection.

**After:**
> The leaf hash covers all tamper-critical fields: structural metadata (subjectId,
> seqNum, entryType, actorId, actorRole, occurredAt, tenancyId, actorType,
> causedByEntryId), base-table supplements (supplementJson), and subclass domain
> content via `domainContentBytes()`.
>
> `canonicalBytes()` is a `public final` instance method on `LedgerEntry` — the
> canonical form is a property of the entry, not of the Merkle tree utility.
> `final` seals the structural encoding; subclasses extend content through
> `domainContentBytes()` only. A build-time guard in `LedgerProcessor` enforces
> this for `@Entity` subclasses with persistent fields.
>
> The save pipeline runs in four phases: content enrichment → hashing → agent
> signing → persist. `AgentEntrySigner` is a direct call, not an enricher —
> signing seals the entry, it does not add content.

Also update the `AgentSignatureSuspectEvent` / enricher pipeline section to
reflect the new architecture (signing extracted from enricher pipeline, content
enrichers only).
