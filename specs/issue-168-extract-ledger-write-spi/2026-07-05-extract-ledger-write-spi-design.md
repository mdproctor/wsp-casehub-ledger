# Extract Ledger Write SPI to ledger-api

**Issue:** casehubio/ledger#168
**Date:** 2026-07-05
**Status:** Design approved

## Problem

`LedgerEntryRepository` (the write interface) lives in `runtime/`, referencing
`runtime.model.LedgerEntry`. Consumers at the api tier — notably blocks#12
(routing accountability) — cannot write ledger entries without depending on
ledger runtime.

The api module has shadow copies of entity classes (`LedgerEntry`,
`LedgerAttestation`, supplements) that are architecturally correct in intent but
never completed: zero consumer references. Everyone uses the runtime types.

Additionally, the runtime `LedgerEntry` bundles three concerns on one class:
data model, verification infrastructure, and JPA entity mapping. This ties
persistence to JPA, violating the platform's SPI standard — the platform
supports H2, PostgreSQL, MongoDB, and Redis.

## Design

### Three-tier model hierarchy

```
api.model.LedgerEntry                        — plain abstract class
                                               consumer-facing fields, supplement helpers
                                               NO JPA annotations, NO persistence coupling

runtime.model.VerifiedLedgerEntry            — extends api.model.LedgerEntry
                                               adds: canonicalBytes(), domainContentBytes()
                                               adds: agentSignature, agentPublicKey, agentKeyRef
                                               adds: actorDid, pendingIdentityStatus
                                               domain logic — persistence-independent

runtime.model.jpa.JpaLedgerEntry             — extends VerifiedLedgerEntry
                                               @Entity, @Inheritance(JOINED), @Table("ledger_entry")
                                               @Column on ALL inherited fields
                                               @NamedQuery, @EntityListeners, @PrePersist
                                               JPA persistence implementation only
```

**Naming rationale:** `VerifiedLedgerEntry` because it adds the verification
envelope — tamper evidence (canonicalBytes for Merkle leaf hash), cryptographic
signing (agent bilateral signing), and identity binding (DID/VC). These are
domain concerns that apply regardless of persistence backend.

Consumer subclasses extend the appropriate tier:

```java
// JPA consumer (engine)
@Entity @Table(name = "case_ledger_entry")
public class CaseLedgerEntry extends JpaLedgerEntry { ... }

// Non-JPA consumer (hypothetical MongoDB)
public class MongoCaseLedgerEntry extends VerifiedLedgerEntry { ... }
```

### LedgerAttestation — two tiers (no verification layer)

Attestations have no agent signing, no canonicalBytes, no DID binding — they
don't need a verification middle tier. Two-tier split:

```
api.model.LedgerAttestation                  — plain class, fields only (remove existing
                                               @MappedSuperclass and @Column annotations)
runtime.model.jpa.JpaLedgerAttestation       — @Entity, @Table, @Column mappings
```

### Same pattern for supplements

```
api.model.supplement.LedgerSupplement        — plain abstract class
api.model.supplement.ComplianceSupplement    — plain class, fields only
api.model.supplement.ProvenanceSupplement    — plain class, fields only
api.model.supplement.LedgerSupplementSerializer — static utility (unchanged)

runtime.model.supplement.*                   — JPA entity versions extending api types
```

### SPI placement

`LedgerEntryRepository` moves to `api/spi/`. Method signatures reference
`api.model.LedgerEntry` and `api.model.LedgerAttestation`:

```java
package io.casehub.ledger.api.spi;

public interface LedgerEntryRepository {
    LedgerEntry save(LedgerEntry entry, String tenancyId);
    LedgerAttestation saveAttestation(LedgerAttestation attestation, String tenancyId);
    List<LedgerEntry> findBySubjectId(UUID subjectId, String tenancyId);
    Optional<LedgerEntry> findLatestBySubjectId(UUID subjectId, String tenancyId);
    Optional<LedgerEntry> findEntryById(UUID id, String tenancyId);
    List<LedgerAttestation> findAttestationsByEntryId(UUID ledgerEntryId, String tenancyId);
    List<LedgerEntry> findBySubjectIdAndTimeRange(UUID subjectId, Instant from, Instant to, String tenancyId);
    List<LedgerEntry> findByActorId(String actorId, Instant from, Instant to, String tenancyId);
    List<LedgerEntry> findByActorRole(String actorRole, Instant from, Instant to, String tenancyId);
    List<LedgerEntry> findCausedBy(UUID entryId, String tenancyId);
    List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(UUID entryId, String capabilityTag, String tenancyId);
    List<LedgerAttestation> findAttestationsByEntryIdGlobal(UUID entryId, String tenancyId);
    List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(String attestorId, String capabilityTag, String tenancyId);
}
```

`ReactiveLedgerEntryRepository` also moves to `api/spi/` — same signatures
with `Uni<T>` returns.

Implementations stay where they are:
- `JpaLedgerEntryRepository` in `runtime/` — works with `JpaLedgerEntry` instances
- `InMemoryLedgerEntryRepository` in `persistence-memory/`
- `NoOpLedgerEntryRepository` in `runtime/`

### Three write paths at the api tier

| SPI | Input type | What it writes | Consumer pattern |
|---|---|---|---|
| `OutcomeRecorder` (existing) | `OutcomeRecord` | LedgerEntry + LedgerAttestation atomically | Decision outcomes with trust-scoring |
| `LedgerAppender` (new) | `AuditRecord` (new) | LedgerEntry only, no attestation | Pure event recording — "this happened" |
| `LedgerEntryRepository` (moved) | `LedgerEntry` subclass | Raw entry/attestation persistence | Consumers with entity subclasses |

#### LedgerAppender

```java
package io.casehub.ledger.api.spi;

public interface LedgerAppender {
    UUID append(AuditRecord record, String tenancyId);
}
```

#### ReactiveledgerAppender

```java
package io.casehub.ledger.api.spi;

public interface ReactiveLedgerAppender {
    Uni<UUID> append(AuditRecord record, String tenancyId);
}
```

#### AuditRecord

```java
package io.casehub.ledger.api.model;

public record AuditRecord(
    UUID subjectId,
    String actorId,
    ActorType actorType,
    String actorRole,
    LedgerEntryType entryType,
    Instant occurredAt,
    UUID causedByEntryId
) {
    public static AuditRecord event(String actorId, UUID subjectId) {
        return new AuditRecord(subjectId, actorId, ActorType.AGENT, null,
            LedgerEntryType.EVENT, null, null);
    }

    public AuditRecord withActorRole(String role) { ... }
    public AuditRecord withCausedBy(UUID entryId) { ... }
    public AuditRecord withOccurredAt(Instant ts) { ... }
}
```

Runtime provides `DefaultLedgerAppender` (`@DefaultBean @ApplicationScoped`):
creates `PlainLedgerEntry` from `AuditRecord`, delegates to
`LedgerEntryRepository.save()`, returns the assigned `id`.

### Dead api model cleanup

**Delete from api (no consumer need at api tier):**
- `api.model.ActorTrustScore` — internal to trust computation. Extract
  `ScoreType` enum to `api.model.ScoreType` standalone.
- `api.model.LedgerMerkleFrontier` — internal to Merkle implementation
- `api.model.ActorIdentity` — internal to pseudonymisation
- `api.model.LedgerEntryArchiveRecord` — internal to retention

### Repositories that stay in runtime

Not consumer-facing — internal to ledger implementation:
- `CrossTenantLedgerEntryRepository` / reactive counterpart
- `KeyRotationRepository` / reactive counterpart
- `ActorIdentityBindingRepository`
- `ErasureReceiptRepository`
- `LedgerMerkleFrontierRepository`
- `ActorTrustScoreRepository`

## Related issues

- **#172** — OutcomeRecord supplementary data (enriches existing write path)
- **#173** — Engine NoOp consolidation (blocked by this issue)
- **#174** — Platform persistence unification (broader pattern this aligns with)

## Breaking changes

This is a breaking change. No deployed instances exist.

- `runtime.model.LedgerEntry` → renamed to `runtime.model.VerifiedLedgerEntry`
- `runtime.model.jpa.JpaLedgerEntry` — new class, extends `VerifiedLedgerEntry`
- `runtime.model.jpa.JpaLedgerAttestation` — new class, extends `api.model.LedgerAttestation`
- Consumer subclass imports change: `extends LedgerEntry` → `extends JpaLedgerEntry`
- `LedgerEntryRepository` package changes: `runtime.repository` → `api.spi`
- `ReactiveLedgerEntryRepository` package changes: `runtime.repository` → `api.spi`
- All JPA annotations removed from api model types
- Engine's 5 `NoOpLedgerEntryRepository` copies need import update (#173)
