# persistence-memory — Design Spec
**Issue:** casehubio/ledger#91
**Date:** 2026-05-23
**Branch:** issue-091-persistence-memory

---

## Overview

Add a `persistence-memory/` module (`casehub-ledger-memory`) providing in-memory
implementations of all ledger persistence SPIs. Two use cases:

1. **Ephemeral install** — evaluate `casehub-ledger` without a database.
2. **Test isolation** — consuming modules test against ledger SPIs without configuring a datasource.

Follows the established `casehub-eidos-memory` pattern: `@Alternative @Priority(1)` beans,
Jandex index in JAR, activates by classpath presence.

Requires three targeted changes to the existing `runtime/` module before the new module
can be correct: frontier SPI extraction, enricher pipeline extraction, and two refactors.

---

## Part 1 — Runtime changes

### 1a. `LedgerMerkleFrontierRepository` SPI

New file: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerMerkleFrontierRepository.java`

```java
public interface LedgerMerkleFrontierRepository {
    List<LedgerMerkleFrontier> findBySubjectId(UUID subjectId);
    void replace(UUID subjectId, List<LedgerMerkleFrontier> newFrontier);
}
```

New JPA impl: `repository/jpa/JpaLedgerMerkleFrontierRepository` — `@Alternative @ApplicationScoped`.
`findBySubjectId` executes `LedgerMerkleFrontier.findBySubjectId` named query.
`replace` executes the delete-then-insert logic currently inline in
`JpaLedgerEntryRepository.updateMerkleFrontier()`.

Activation: same pattern as `JpaLedgerEntryRepository` — `@Alternative`, selected via
`quarkus.arc.selected-alternatives` or subclassing. Consumers who activate
`JpaLedgerEntryRepository` must also activate `JpaLedgerMerkleFrontierRepository`.

### 1b. Refactor `JpaLedgerEntryRepository`

- Inject `LedgerMerkleFrontierRepository frontierRepo`.
- `updateMerkleFrontier()` calls `frontierRepo.findBySubjectId()` then `frontierRepo.replace()`.
- Removes direct EntityManager usage for frontier operations (entry queries are unchanged).

### 1c. Refactor `LedgerVerificationService`

- Inject `LedgerMerkleFrontierRepository frontierRepo` instead of `@LedgerPersistenceUnit EntityManager em`.
- `treeRoot()` calls `frontierRepo.findBySubjectId()`.
- `verify()` unchanged — recomputes frontier locally, calls `treeRoot()` at the end.
- `EntityManager` field removed from this class entirely.

### 1d. Extract `LedgerEnricherPipeline`

New file: `service/LedgerEnricherPipeline.java` — `@ApplicationScoped`.

```java
@ApplicationScoped
public class LedgerEnricherPipeline {
    @Inject Instance<LedgerEntryEnricher> enrichers;

    public void enrich(LedgerEntry entry) {
        for (LedgerEntryEnricher enricher : enrichers) {
            try { enricher.enrich(entry); } catch (Exception e) { /* non-fatal */ }
        }
    }
}
```

`LedgerTraceListener.prePersist()` delegates to `LedgerEnricherPipeline.enrich()`.
No change to `LedgerEntryEnricher` SPI or any enricher implementations.

---

## Part 2 — `persistence-memory/` module

**Artifact:** `casehub-ledger-memory`
**Package:** `io.casehub.ledger.memory`
**Parent:** `casehub-ledger-parent`

### Maven dependencies

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>  <!-- runtime module — SPIs + CDI beans -->
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-arc</artifactId>
</dependency>
<!-- test scope: quarkus-junit5, assertj-core -->
```

Jandex index generated at build time via `jandex-maven-plugin` — required for
`@Alternative` bean discovery from JAR without copying sources (GE-20260420-4a62d3).
Consumers add as compile-scope dependency (not test-scope) for `@QuarkusTest` CDI discovery.

### `InMemoryLedgerEntryRepository`

`@Alternative @Priority(1) @ApplicationScoped`

Fields:
- `ConcurrentHashMap<UUID, LedgerEntry> entries`
- `ConcurrentHashMap<UUID, LedgerAttestation> attestations`
- `ConcurrentHashMap<UUID, AtomicLong> sequenceCounters`
- `@Inject LedgerMerkleFrontierRepository frontierRepo`
- `@Inject LedgerEnricherPipeline enricherPipeline`
- `@Inject ActorIdentityProvider actorIdentityProvider`
- `@Inject DecisionContextSanitiser decisionContextSanitiser`
- `@Inject LedgerMerklePublisher merklePublisher`
- `@Inject LedgerConfig ledgerConfig`

`save()` pipeline (mirrors `JpaLedgerEntryRepository.save()`):
1. Set `occurredAt = Instant.now()` if null
2. Tokenise `actorId` via `actorIdentityProvider.tokenise()`
3. Sanitise `decisionContext` in any attached `ComplianceSupplement`
4. Call `enricherPipeline.enrich(entry)` — runs trace ID, agent signature, and any custom enrichers
5. Compute `entry.digest = LedgerMerkleTree.leafHash(entry)` if hash chain enabled
6. Assign `entry.id = UUID.randomUUID()` if null
7. Assign `entry.sequenceNumber` from per-subject atomic counter
8. Store in `entries` map keyed by `entry.id`
9. Call `frontierRepo.replace(subjectId, newFrontier)` if hash chain enabled
10. Call `merklePublisher.publish()`
11. Return entry

All query methods are stream filters over `entries.values()` and `attestations.values()`.
`findAttestationsForEntries` groups by `ledgerEntryId`. Ordering matches JPA (`occurredAt` or
`sequenceNumber` ascending per method contract).

`clear()` — resets all three maps and counters. Called in `@BeforeEach` in tests.

### `InMemoryLedgerMerkleFrontierRepository`

`@Alternative @Priority(1) @ApplicationScoped`

Field: `ConcurrentHashMap<UUID, List<LedgerMerkleFrontier>> frontierBySubject`

- `findBySubjectId` — returns defensive copy (`List.copyOf`) or empty list
- `replace` — `frontierBySubject.put(subjectId, List.copyOf(newFrontier))`
- `clear()` — resets map

### `InMemoryActorTrustScoreRepository`

`@Alternative @Priority(1) @ApplicationScoped`

Field: `ConcurrentHashMap<String, ActorTrustScore> store`

Key format: `actorId + "|" + scoreType + "|" + nvl(capabilityKey) + "|" + nvl(dimensionKey)`
where `nvl(x)` = `x != null ? x : ""`

- `upsert` — constructs or replaces `ActorTrustScore` at the composite key
- `updateGlobalTrustScore` — finds the GLOBAL row by `actorId|GLOBAL||` key, updates field in place
- All `findBy*` methods stream-filter the map values
- `findAll`, `findAllByLastComputedAtAfter` — full scan with optional time filter
- `clear()` — resets map

### `InMemoryKeyRotationRepository`

`@Alternative @Priority(1) @ApplicationScoped`

Injects `InMemoryLedgerEntryRepository blocking` (concrete type — shares the same CDI singleton).

- `findByActorId` — filters `blocking.entries.values()` for `KeyRotationEntry` instances with matching `actorId`, sorted by `occurredAt` ascending
- `findCompromisedByActorIdAndKeyRef` — additionally filters `reason == COMPROMISED` and `keyRef` matches

No separate state. Single source of truth is the entries map.
`entries` field on `InMemoryLedgerEntryRepository` is package-private (not `private`) so
this class can access it when injected as the concrete type.

### `InMemoryReactiveLedgerEntryRepository`

`@Alternative @Priority(1) @ApplicationScoped`
`@IfBuildProperty(name="casehub.ledger.reactive.enabled", stringValue="true")`

Injects `InMemoryLedgerEntryRepository blocking`. Every method wraps the blocking call:
```java
public Uni<LedgerEntry> save(LedgerEntry entry) {
    return Uni.createFrom().item(() -> blocking.save(entry));
}
```

Shared state: CDI injects the same `InMemoryLedgerEntryRepository` singleton — test setup
via `blocking.save()` is visible to reactive service code via this class (GE-20260521-49e7fd).

### `InMemoryReactiveKeyRotationRepository`

`@Alternative @Priority(1) @ApplicationScoped`
`@IfBuildProperty(name="casehub.ledger.reactive.enabled", stringValue="true")`

Injects `InMemoryKeyRotationRepository blocking`. Wraps both methods in `Uni`.

---

## Part 3 — Testing

Test module: `persistence-memory/src/test/`

`src/test/resources/application.properties`:
```properties
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository
```

### `InMemoryLedgerEntryRepositoryTest` (`@QuarkusTest`)

`@Inject InMemoryLedgerEntryRepository repository` (concrete type for `clear()`).
`@BeforeEach repository.clear()`.

Covers: `save` assigns fields correctly; `findBySubjectId` ordered; `findLatestBySubjectId`;
`findEntryById`; `findByTimeRange`; `findByActorId`; `findByActorRole`; `findCausedBy`;
`findAllEvents`; `listAll`; `saveAttestation` + all attestation query methods;
`findAttestationsForEntries` grouping.

### `InMemoryActorTrustScoreRepositoryTest` (`@QuarkusTest`)

Covers all four `ScoreType` combinations; `upsert` idempotency; `updateGlobalTrustScore`;
`findAll`; `findAllByLastComputedAtAfter`.

### `InMemoryKeyRotationRepositoryTest` (`@QuarkusTest`)

Seeds `KeyRotationEntry` via `InMemoryLedgerEntryRepository.save()`.
Verifies `findByActorId` ordering; `findCompromisedByActorIdAndKeyRef` filter.

Reactive delegates are not separately tested — covered by the existing ArchUnit
blocking/reactive parity test in the `runtime` module.

---

## Part 4 — Platform and project updates

**`casehub-ledger-parent/pom.xml`** — add `<module>persistence-memory</module>`.

**`PLATFORM.md`** (casehub-parent):
- Capability Ownership: new row — "In-memory persistence (zero datasource / ephemeral install)" → `casehub-ledger` → `casehub-ledger-memory`, `InMemory*` impls
- Repository Map `casehub-ledger` description: append `persistence-memory/ (casehub-ledger-memory)`

**`CLAUDE.md`** (this project) — add `persistence-memory/` to project structure.

---

## Out of Scope

- `ReactiveActorTrustScoreRepository` in-memory impl — SPI does not exist yet
- Changes to deployment module's `LedgerProcessor` — reactive gating uses `@IfBuildProperty` directly
- `BlockingReactiveKeyRotationRepository` shim in `runtime/src/test/` — unrelated JPA-backed test shim, stays as-is
- New named queries or schema changes — no Flyway involvement
