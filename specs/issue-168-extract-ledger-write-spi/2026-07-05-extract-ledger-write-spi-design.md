# Extract Ledger Write SPI to ledger-api

**Issue:** casehubio/ledger#168
**Date:** 2026-07-05
**Status:** Design approved

## Problem

`LedgerEntryRepository` (the write interface) lives in `runtime/`, referencing
`runtime.model.LedgerEntry`. Consumers at the api tier cannot write ledger
entries without depending on ledger runtime.

The api module has shadow copies of entity classes (`LedgerEntry`,
`LedgerAttestation`, supplements) that are architecturally correct in intent but
never completed: zero consumer references. Everyone uses the runtime types.

Additionally, the runtime `LedgerEntry` bundles two concerns on one class:
data model (fields, supplement helpers, canonical bytes) and JPA entity mapping
(`@Entity`, `@Inheritance`, `@NamedQuery`, `@PrePersist`, `@EntityListeners`).
Separating these concerns follows the pattern already established by
`LedgerAttestation`, where `api.model.LedgerAttestation` is `@MappedSuperclass`
and `runtime.model.LedgerAttestation` is the `@Entity`.

## Design

### Two-tier model hierarchy

```
api.model.LedgerEntry                        — @MappedSuperclass
                                               ALL persistent fields with @Column
                                               (core, signing, DID, supplementJson)
                                               @Transient supplements list + helpers
                                               canonicalBytes(), domainContentBytes()
                                               NO @Entity, NO @Inheritance, NO @Table
                                               NO @NamedQuery, NO @PrePersist

runtime.model.jpa.JpaLedgerEntry             — extends api.model.LedgerEntry
                                               @Entity(name = "LedgerEntry"), @Inheritance(JOINED)
                                               @Table("ledger_entry"), @DiscriminatorColumn
                                               @NamedQuery, @EntityListeners, @PrePersist
                                               @Transient pendingIdentityStatus
                                               @OneToMany for JPA supplement persistence
```

**Design rationale:** This follows the existing `LedgerAttestation` pattern,
where `api.model.LedgerAttestation` is `@MappedSuperclass` with `@Column`
annotations and `runtime.model.LedgerAttestation` adds `@Entity`, `@Table`,
`@NamedQuery`, `@PrePersist`. The api module already depends on
`jakarta.persistence-api` (provided scope) for exactly this purpose.

All persistent fields — including agent signing (`agentSignature`,
`agentPublicKey`, `agentKeyRef`), DID binding (`actorDid`), and hash chain
(`digest`) — belong on `api.model.LedgerEntry` because they are persisted
columns on every entry. `canonicalBytes()` and `domainContentBytes()` also
belong here: they compute over the data model fields and are needed by any
persistence backend that maintains tamper evidence.

The `@Transient pendingIdentityStatus` field stays on `JpaLedgerEntry` — it
is a runtime implementation detail (set by an enricher, read by an entity
listener, never persisted).

**Supplement persistence:** `api.model.LedgerEntry` declares supplements as
`@Transient` (for the api-tier helper methods). `JpaLedgerEntry` adds an
internal `@OneToMany` field for JPA cascade persistence. `JpaLedgerEntry`
overrides `attach()` to synchronise both lists, and uses `@PostLoad` to
populate the transient list on entity load.

**ID assignment:** `api.model.LedgerEntry` constructor eagerly assigns
`id = UUID.randomUUID()`. This ensures non-JPA backends have an ID without
requiring JPA lifecycle callbacks. `JpaLedgerEntry`'s `@PrePersist` remains
as a defensive fallback.

**JPQL compatibility:** `@Entity(name = "LedgerEntry")` on `JpaLedgerEntry`
preserves the entity name used in all `@NamedQuery` JPQL (`FROM LedgerEntry e`).

Consumer subclasses extend the appropriate tier:

```java
// JPA consumer (engine)
@Entity @Table(name = "case_ledger_entry")
public class CaseLedgerEntry extends JpaLedgerEntry { ... }

// Non-JPA consumer (hypothetical MongoDB)
public class MongoCaseLedgerEntry extends LedgerEntry { ... }
```

### Internal subclasses requiring reparenting

All production subclasses currently extending `runtime.model.LedgerEntry`
must reparent to `runtime.model.jpa.JpaLedgerEntry`:

| Class | File | Notes |
|---|---|---|
| `PlainLedgerEntry` | `runtime/model/PlainLedgerEntry.java` | @Entity, @Table, @DiscriminatorValue |
| `KeyRotationEntry` | `runtime/model/KeyRotationEntry.java` | @Entity, @Table, @DiscriminatorValue, domain fields |
| `ErasureReceiptLedgerEntry` | `runtime/model/ErasureReceiptLedgerEntry.java` | @Entity, @Table, @DiscriminatorValue, domain fields |
| `ActorIdentityBindingEntry` | `runtime/model/ActorIdentityBindingEntry.java` | @Entity, @Table, @DiscriminatorValue, domain fields |

Test subclasses across runtime and persistence-memory tests also need
reparenting (TestLedgerEntry, ConcreteEntry, MemoryTestEntry, etc.).

### LedgerAttestation — keep existing two-tier pattern

`api.model.LedgerAttestation` already uses `@MappedSuperclass` with `@Id` and
`@Column` annotations. `runtime.model.LedgerAttestation` extends it with
`@Entity`, `@Table`, `@NamedQuery`, `@PrePersist`. This pattern is correct
and unchanged — no annotations are removed.

### Supplements — already done

The api supplement classes (`LedgerSupplement`, `ComplianceSupplement`,
`ProvenanceSupplement`) are already plain POJOs with zero JPA annotations.
The runtime supplement classes already extend them with `@Entity`, `@Table`,
`@Column` annotations. No changes needed to the supplement hierarchy.

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

#### ReactiveLedgerAppender

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
    public AuditRecord {
        if (entryType == LedgerEntryType.ATTESTATION) {
            throw new IllegalArgumentException(
                "AuditRecord does not support ATTESTATION — use OutcomeRecorder for attestations");
        }
    }

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

### NoOp implementations

Platform convention requires `@DefaultBean` no-op implementations for all
store SPIs. New no-ops in `runtime/`:

- `NoOpLedgerAppender` — returns `UUID.randomUUID()` without persisting
- `NoOpReactiveLedgerAppender` — returns `Uni.createFrom().item(UUID.randomUUID())`

Existing no-ops (`NoOpLedgerEntryRepository`, `NoOpReactiveLedgerEntryRepository`)
need import updates after the interface moves to `api/spi/`.

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

### persistence-memory module updates

The following classes in `persistence-memory/` implement `LedgerEntryRepository`
or `ReactiveLedgerEntryRepository` and need import updates after the interface
moves to `api/spi/`:

- `InMemoryLedgerEntryRepository`
- `InMemoryReactiveLedgerEntryRepository`
- `InMemoryCrossTenantLedgerEntryRepository`
- `InMemoryCrossTenantReactiveLedgerEntryRepository`

Test subclasses (e.g. `MemoryTestEntry`) need reparenting.

## Related issues

- **#172** — OutcomeRecord supplementary data (enriches existing write path)
- **#173** — Engine NoOp consolidation (blocked by this issue)
- **#174** — Platform persistence unification (broader pattern this aligns with)

## Breaking changes

This is a breaking change. No deployed instances exist.

- `runtime.model.LedgerEntry` → split: fields and logic promoted to
  `api.model.LedgerEntry` (`@MappedSuperclass`), JPA entity moved to
  `runtime.model.jpa.JpaLedgerEntry` (`@Entity`)
- Consumer subclass imports change: `extends LedgerEntry` → `extends JpaLedgerEntry`
- Internal subclasses reparented: `PlainLedgerEntry`, `KeyRotationEntry`,
  `ErasureReceiptLedgerEntry`, `ActorIdentityBindingEntry`
- `LedgerEntryRepository` package changes: `runtime.repository` → `api.spi`
- `ReactiveLedgerEntryRepository` package changes: `runtime.repository` → `api.spi`
- `LedgerEntryEnricher.enrich()` parameter type changes:
  `runtime.model.LedgerEntry` → `api.model.LedgerEntry`
- Engine's 5 `NoOpLedgerEntryRepository` copies need import update (#173)
- ARC42STORIES.MD §5 Key Files: update `LedgerEntry.java` and
  `LedgerEntryRepository.java` references; update §5.3.1 save pipeline
  description; update Layer Taxonomy L1 repository context
