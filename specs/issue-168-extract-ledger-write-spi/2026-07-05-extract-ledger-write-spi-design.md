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
`@Transient` (for the api-tier helper methods). `JpaLedgerEntry` adds
separate `@OneToMany` fields per supplement type (`complianceSupplements`,
`provenanceSupplements`) for JPA cascade persistence. `JpaLedgerEntry`
overrides `attach()` to synchronise both the transient list and the
appropriate JPA list, and uses `@PostLoad` to populate the transient list
from the JPA lists on entity load. See §Supplements for full details.

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

### Supplements — two-tier refactoring

The api supplement classes (`LedgerSupplement`, `ComplianceSupplement`,
`ProvenanceSupplement`) and their runtime counterparts are **parallel copies**
with zero inheritance between them. Unlike `LedgerAttestation` (where
`runtime.model.LedgerAttestation extends api.model.LedgerAttestation`), the
supplement hierarchies are completely independent type trees. This must be
fixed — without it, the two-tier `LedgerEntry` design does not compile.

**Concrete failures without this fix:**
1. `ProvenanceCaptureEnricher` creates `runtime.model.supplement.ProvenanceSupplement`
   and calls `entry.attach(ps)`. After the two-tier refactoring, `attach()` on
   `api.model.LedgerEntry` accepts `api.model.supplement.LedgerSupplement` — type error.
2. `api.model.LedgerEntry.compliance()` uses `instanceof api.ComplianceSupplement`,
   but JPA loads `runtime.ComplianceSupplement` — `instanceof` returns false.
3. The `ledgerEntry` back-reference on runtime supplements points to
   `runtime.model.LedgerEntry`, which no longer exists after the rename.

**Design:** Each supplement type follows the two-tier pattern independently.
Api classes become `@MappedSuperclass` with `@Column` annotations. Runtime
classes extend their api counterparts as `@Entity`:

```
api.model.supplement.LedgerSupplement         — @MappedSuperclass
                                                @Id on id, @Column on supplementType
                                                @Transient ledgerEntry (in-memory back-ref)

api.model.supplement.ComplianceSupplement     — @MappedSuperclass
                                                extends api.LedgerSupplement
                                                @Column on all compliance fields

api.model.supplement.ProvenanceSupplement     — @MappedSuperclass
                                                extends api.LedgerSupplement
                                                @Column on all provenance fields

runtime.model.supplement.JpaComplianceSupplement — @Entity
                                                    @Table("ledger_supplement_compliance")
                                                    extends api.ComplianceSupplement
                                                    @ManyToOne + @JoinColumn for ledgerEntry
                                                    @PrePersist for ID assignment

runtime.model.supplement.JpaProvenanceSupplement — @Entity
                                                    @Table("ledger_supplement_provenance")
                                                    extends api.ProvenanceSupplement
                                                    @ManyToOne + @JoinColumn for ledgerEntry
                                                    @PrePersist for ID assignment
```

**JOINED inheritance is eliminated.** The current supplements use
`@Inheritance(JOINED)` with a shared `ledger_supplement` base table and
joined subtables. This is incompatible with the two-tier pattern: Java
single inheritance prevents runtime supplements from both extending their
api counterparts (required for `instanceof` type safety in `compliance()`
and `provenance()` helpers) and sharing a common `@Entity` root (required
for JOINED inheritance). Type safety is the non-negotiable constraint —
`instanceof` checks are part of the api contract. JOINED inheritance is a
persistence implementation detail. Each supplement type becomes its own
independent entity with a self-contained table.

**Schema change:** The current three-table schema (`ledger_supplement` +
joined `ledger_supplement_compliance` + joined `ledger_supplement_provenance`)
collapses to two self-contained tables. Each carries the base columns (`id`,
`ledger_entry_id`, `supplement_type`) plus its type-specific columns. No
deployed instances exist, so this is a clean schema change.

**JpaLedgerEntry integration:** The single polymorphic
`@OneToMany List<LedgerSupplement>` is replaced with separate per-type
relationships on `JpaLedgerEntry`:
- `@OneToMany List<JpaComplianceSupplement> complianceSupplements`
- `@OneToMany List<JpaProvenanceSupplement> provenanceSupplements`

The transient `supplements` list (inherited from `api.model.LedgerEntry`)
is populated by `@PostLoad` from these JPA lists. `attach()` is overridden
to synchronise both the transient list and the appropriate JPA list.

**Serializer unification:** The duplicate `LedgerSupplementSerializer` in
`runtime.model.supplement` is deleted. The api copy becomes canonical.

**`runtime.model.supplement.LedgerSupplement` is deleted.** No runtime
abstract supplement base is needed — each runtime supplement entity directly
extends its api counterpart.

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

Existing `NoOpLedgerEntryRepository` needs import updates after the interface
moves to `api/spi/`.

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
- `LedgerProcessor.LEDGER_ENTRY` DotName: `io.casehub.ledger.runtime.model.LedgerEntry`
  → `io.casehub.ledger.api.model.LedgerEntry`. Without this update, the
  `validateLedgerEntryFieldShadowing` and `validateDomainContentBytes` build-time
  validators silently stop finding subclasses (empty `getAllKnownSubclasses` result).
  The `@MappedSuperclass` base correctly catches all subclasses; the
  `domainContentBytes` validator already filters on `@Entity`.
- Supplement hierarchy: parallel copies unified into two-tier pattern. JOINED
  inheritance (`ledger_supplement` base table) eliminated — each supplement type
  becomes an independent entity with a self-contained table. `runtime.model.supplement.LedgerSupplement` deleted.
  Runtime serializer deleted (api copy becomes canonical).
- ARC42STORIES.MD §5 Key Files: update `LedgerEntry.java` and
  `LedgerEntryRepository.java` references; update §5.3.1 save pipeline
  description; update Layer Taxonomy L1 repository context
