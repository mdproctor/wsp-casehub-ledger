# Trust Federation and Bootstrap — Design Spec
*Date: 2026-05-12*
*Issues: #63 (C1), #64 (C2), #65 (D1) — Epics #51 (Group C), #52 (Group D)*

---

## Scope

Group C adds a structured read-model over `ActorTrustScore` (`TrustExportService`) and a pluggable import SPI (`TrustImportService`) so upper layers and future deployments can consume trust data. Group D adds a bootstrap hook in `TrustScoreJob` that seeds first-time actors from an external source rather than starting from the uninformative Beta(1,1) prior.

**Explicitly out of scope:** REST endpoint, `HttpTrustBootstrapSource`. Both presuppose a multi-deployment topology that does not yet exist. The SPIs are designed to receive them as follow-ons.

---

## Data Model: `TrustExportPayload`

All types are plain Java records in `runtime/service/federation/`.

```
TrustExportPayload
  exportedAt: Instant
  exportingDeployment: String          // from config; empty string if not set
  actors: List<ActorExport>

ActorExport
  actorId: String
  actorType: ActorType
  globalScore: GlobalScoreExport       // null if no GLOBAL row computed yet
  capabilityScores: List<CapabilityScoreExport>
  dimensionScores: List<DimensionScoreExport>

GlobalScoreExport
  alpha: double
  beta: double
  trustScore: double
  decisionCount: int
  attestationPositive: int
  attestationNegative: int
  lastComputedAt: Instant

CapabilityScoreExport
  capabilityTag: String
  alpha: double
  beta: double
  trustScore: double
  decisionCount: int
  attestationPositive: int
  attestationNegative: int
  lastComputedAt: Instant

DimensionScoreExport
  dimension: String
  score: double
  sampleCount: int                     // attestationPositive + attestationNegative
  lastComputedAt: Instant
```

DIMENSION rows carry only a continuous `score` and no alpha/beta — their shape is deliberately different from GLOBAL/CAPABILITY rows, reflecting a genuinely different kind of signal. `sampleCount` is derived from `attestationPositive + attestationNegative` on the source `ActorTrustScore` row.

---

## Group C — Trust Federation

### C1: `TrustExportService`

CDI bean in `runtime/service/federation/`. Reads from `ActorTrustScoreRepository` and projects rows into the structured export format.

**Methods:**

```java
TrustExportPayload exportAll(double minTrustScore);
Optional<TrustExportPayload> exportActor(String actorId);
TrustExportPayload exportDelta(Instant since);
```

- `exportAll`: loads all rows via `findAll()`, groups by actorId, filters actors whose GLOBAL `trustScore >= minTrustScore`. Actors with no GLOBAL row are excluded.
- `exportActor`: loads rows for a single actor via `findByActorIdAndScoreType()` for each type. Returns empty if the actor has no scores.
- `exportDelta`: loads actors whose `lastComputedAt` is after `since` via a new repository method.

**New `ActorTrustScoreRepository` method:**

```java
List<ActorTrustScore> findAllByLastComputedAtAfter(Instant since);
```

Backed by a `@NamedQuery` on `ActorTrustScore`. The `ActorTrustScoreRepository` is not a `LedgerEntryRepository` mirror — adding this method does not trigger the ledger-SPI-propagation protocol.

**New config key:**

```
casehub.ledger.trust.export.deployment-id=   # optional, empty default
```

Added to `TrustScoreConfig` as `ExportConfig` nested interface.

---

### C2: `TrustImportService`

SPI in `runtime/service/federation/`. Single method — the implementation is the strategy.

```java
public interface TrustImportService {
    void importTrust(TrustExportPayload payload);
}
```

**`NoOpTrustImportService`** — `@DefaultBean @ApplicationScoped`. Empty body. Zero behaviour by default.

**`JpaTrustImportService`** — `@Alternative @Priority(1) @ApplicationScoped`. Seed-if-absent for all row types.

Algorithm:
1. For each `ActorExport` in `payload.actors()`:
2. Check `trustRepo.findByActorId(actorId)` — if a GLOBAL row exists, skip the entire actor.
3. If no GLOBAL row: write `globalScore` as GLOBAL row, each `capabilityScores` entry as CAPABILITY rows, each `dimensionScores` entry as DIMENSION rows via `trustRepo.upsert()`.

One `findByActorId` call per actor, no per-row queries. DIMENSION rows are seeded identically to GLOBAL and CAPABILITY — the strategy is uniformly seed-if-absent regardless of score type. Custom merge behaviour (weighted average, replace, DIMENSION-specific logic) is the province of custom `TrustImportService` implementations.

Activation:
```properties
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.service.federation.JpaTrustImportService
```

---

## Group D — Trust Bootstrapping

### D1: `TrustBootstrapSource` SPI

```java
public interface TrustBootstrapSource {
    Optional<TrustExportPayload> fetchPriorTrust(String actorId);
}
```

**`NoOpTrustBootstrapSource`** — `@DefaultBean @ApplicationScoped`. Returns `Optional.empty()`.

---

### `TrustBootstrapService`

CDI bean in `runtime/service/federation/`. Called from `TrustScoreJob`.

```java
@ApplicationScoped
public class TrustBootstrapService {

    @Inject TrustBootstrapSource bootstrapSource;
    @Inject TrustImportService importService;

    public void bootstrapIfNew(Set<String> newActorIds) {
        for (String actorId : newActorIds) {
            bootstrapSource.fetchPriorTrust(actorId)
                .ifPresent(importService::importTrust);
        }
    }
}
```

Silently skips actors for which the source returns empty. Failures in `fetchPriorTrust` propagate — callers should handle or log.

---

### Hook in `TrustScoreJob`

Added at the start of `runComputation()`, before the per-actor loop, gated by `casehub.ledger.trust.bootstrap.enabled`:

```java
if (config.trustScore().bootstrap().enabled()) {
    Set<String> existingActors = trustRepo.findAll().stream()
        .map(s -> s.actorId)
        .collect(Collectors.toSet());
    Set<String> newActors = new LinkedHashSet<>(byActor.keySet());
    newActors.removeAll(existingActors);
    if (!newActors.isEmpty()) {
        bootstrapService.bootstrapIfNew(newActors);
    }
}
```

`findAll()` is already called later in `runComputation()` for routing signals — the pre-pass adds one extra call on job runs where bootstrap is enabled. Bootstrap fires once per actor per deployment: once `JpaTrustImportService` writes a GLOBAL row, subsequent runs find it and skip.

**New config keys:**

```
casehub.ledger.trust.bootstrap.enabled=false   # opt-in; default off
```

Added as `BootstrapConfig` nested interface in `LedgerConfig.TrustScoreConfig`.

---

## Package Layout

All new types under `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/`:

```
federation/
  TrustExportPayload.java
  ActorExport.java
  GlobalScoreExport.java
  CapabilityScoreExport.java
  DimensionScoreExport.java
  TrustExportService.java
  TrustImportService.java
  NoOpTrustImportService.java
  JpaTrustImportService.java
  TrustBootstrapSource.java
  NoOpTrustBootstrapSource.java
  TrustBootstrapService.java
```

---

## Testing

### `TrustExportServiceTest` — `@QuarkusTest`

- `exportAll(0.0)` returns all actors with computed scores
- `exportAll(threshold)` excludes actors below threshold
- `exportAll` excludes actors with no GLOBAL row
- `exportActor` returns structured payload with global, capability, and dimension sections populated
- `exportActor` returns empty for unknown actorId
- `exportDelta(since)` returns only actors whose `lastComputedAt` is after `since`
- `exportDelta(since)` returns empty payload when no scores changed since `since`

### `TrustImportServiceTest` — `@QuarkusTest`

- New actor → all rows seeded (GLOBAL, CAPABILITY, DIMENSION)
- Existing actor (GLOBAL row present) → entire actor skipped, no rows written
- Mixed payload (one new, one existing) → only new actor seeded
- `NoOpTrustImportService` → no rows written regardless of payload

### `TrustBootstrapServiceTest` — `@QuarkusTest`

- New actor + source returns payload → rows seeded via import service
- New actor + source returns empty → no rows written
- Existing actor → not passed to `bootstrapIfNew` (filtered by pre-pass)
- Bootstrap disabled via config → pre-pass skipped entirely

### `TrustBootstrapSourceTest` — plain JUnit 5 (no Quarkus)

- `NoOpTrustBootstrapSource.fetchPriorTrust(anyActorId)` returns `Optional.empty()`

---

## Protocol Compliance

- No changes to `LedgerEntryRepository` or `ReactiveLedgerEntryRepository` — ledger-SPI-propagation protocol does not apply.
- New `ActorTrustScoreRepository` method (`findAllByLastComputedAtAfter`) has no downstream consumer implementations — no propagation needed.
- All SPIs follow the `@DefaultBean` no-op pattern established by `GlobalScoreStrategy`, `DecayFunction`, and `LedgerReconciliationSource`.
- Config keys follow the `casehub.ledger.*` prefix convention.
