# CAPABILITY_DIMENSION Composite Trust Score — Design Spec

**Issue:** casehubio/ledger#76  
**Date:** 2026-05-14  
**Status:** Approved

---

## Problem

`ActorTrustScore` has three score types: `GLOBAL`, `CAPABILITY`, and `DIMENSION`. A
`LedgerAttestation` can carry both `capabilityTag` and `trustDimension` simultaneously,
but the two signals are stored independently — the composite is lost.

An agent who is meticulous at security review but sloppy at architecture review looks
identical to one who is uniformly mediocre at both. When routing a high-stakes security
PR, the question "is this agent thorough specifically for security reviews?" is
unanswerable with today's model.

---

## Solution Overview

Add `CAPABILITY_DIMENSION` as a fourth score type — one row per
`(actor, capability tag, quality dimension)`. The schema is extended with two explicit
nullable columns replacing the single `scope_key` column. Every layer from schema to
federation export is updated consistently.

---

## 1. Schema — V1005

Drop `scope_key`. Replace with two typed nullable columns:

```sql
capability_key  VARCHAR(255)   -- null unless capability-scoped
dimension_key   VARCHAR(255)   -- null unless dimension-scoped
```

Score type semantics fall out from which columns are non-null:

| score_type           | capability_key      | dimension_key       |
|----------------------|---------------------|---------------------|
| GLOBAL               | null                | null                |
| CAPABILITY           | "security-review"   | null                |
| DIMENSION            | null                | "thoroughness"      |
| CAPABILITY_DIMENSION | "security-review"   | "thoroughness"      |

`score_type` is retained as an explicit discriminator — it enables clean indexed queries
(`WHERE score_type = 'CAPABILITY'`) without multi-column IS NULL expressions everywhere.

**Unique constraint** — `score_type` drops out because the type is now deterministic
from the two key columns:

```sql
CONSTRAINT uq_actor_trust_score_key
    UNIQUE NULLS NOT DISTINCT (actor_id, capability_key, dimension_key)
```

**CHECK constraint** — makes the schema self-enforcing:

```sql
CONSTRAINT chk_actor_trust_score_keys CHECK (
    (score_type = 'GLOBAL'               AND capability_key IS NULL     AND dimension_key IS NULL    ) OR
    (score_type = 'CAPABILITY'           AND capability_key IS NOT NULL  AND dimension_key IS NULL    ) OR
    (score_type = 'DIMENSION'            AND capability_key IS NULL      AND dimension_key IS NOT NULL) OR
    (score_type = 'CAPABILITY_DIMENSION' AND capability_key IS NOT NULL  AND dimension_key IS NOT NULL)
)
```

No encoding. No LIKE queries. The schema says exactly what it means.

---

## 2. API Module (`casehub-ledger-api`)

### `ScoreType` enum

```java
public enum ScoreType {
    GLOBAL,
    CAPABILITY,
    DIMENSION,
    CAPABILITY_DIMENSION   // one row per (actor, capability tag, quality dimension)
}
```

### `ActorTrustScore` (`@MappedSuperclass`)

Drop `scopeKey`. Add two typed fields:

```java
/** Capability tag for CAPABILITY and CAPABILITY_DIMENSION rows; null otherwise. */
@Column(name = "capability_key")
public String capabilityKey;

/** Quality dimension name for DIMENSION and CAPABILITY_DIMENSION rows; null otherwise. */
@Column(name = "dimension_key")
public String dimensionKey;
```

Javadoc on the class documents all four score types and their key combinations explicitly.

---

## 3. Repository SPI (`ActorTrustScoreRepository`)

### Remove

`findByActorIdAndTypeAndKey(String actorId, ScoreType scoreType, String scopeKey)` —
the generic method that forced callers to understand `scope_key` semantics per type.

### Add typed read methods

```java
/** CAPABILITY row for this actor and capability tag. */
Optional<ActorTrustScore> findCapabilityScore(String actorId, String capabilityTag);

/** DIMENSION row for this actor and dimension name. */
Optional<ActorTrustScore> findDimensionScore(String actorId, String dimension);

/** CAPABILITY_DIMENSION row for this actor, capability tag, and dimension. */
Optional<ActorTrustScore> findCapabilityDimension(String actorId, String capabilityTag, String dimension);

/** All CAPABILITY_DIMENSION rows for this actor and capability tag. */
List<ActorTrustScore> findCapabilityDimensions(String actorId, String capabilityTag);
```

### Keep unchanged

`findByActorId(actorId)` (GLOBAL), `findByActorIdAndScoreType(actorId, scoreType)`,
`findAll()`, `findAllByLastComputedAtAfter(since)`, `updateGlobalTrustScore(actorId, score)`.

### Update upsert signature

Replace single `scopeKey` with two typed nullable parameters:

```java
void upsert(String actorId, ScoreType scoreType,
            String capabilityKey, String dimensionKey,
            ActorType actorType, double trustScore,
            int decisionCount, int overturnedCount,
            double alpha, double beta,
            int attestationPositive, int attestationNegative,
            Instant lastComputedAt);
```

### Named queries

All named queries on `ActorTrustScore` are updated to use `capabilityKey` /
`dimensionKey` predicates. `findCapabilityDimensions` uses:

```sql
WHERE a.actorId = :actorId
  AND a.scoreType = 'CAPABILITY_DIMENSION'
  AND a.capabilityKey = :capabilityKey
```

No LIKE patterns needed.

### Implementations to update

- `JpaActorTrustScoreRepository` — production implementation
- `StubRepository` in `TrustGateServiceTest` — test stub (returns `Optional.empty()` /
  `List.of()` for new methods)

---

## 4. `TrustScoreJob` — CAPABILITY_DIMENSION Pass

A fourth pass slots between the existing dimension pass and the global pass. It reuses
`actorAttestations` (already loaded) and `computeDimensionScore` (already exists) —
no new queries, no new statistical logic.

**Filter:** attestations where:
- `capabilityTag != null && !CapabilityTag.GLOBAL`
- `trustDimension != null`

These are a strict subset of the attestations the dimension pass sees.

**Group by `(capabilityTag, trustDimension)`** — a two-key grouping, producing one
score per unique pair.

**Compute:** `computeDimensionScore(group, now)` — same decay-weighted average as the
DIMENSION pass. Same metrics shape: `alpha=0.0`, `beta=0.0`, positive/negative counts,
`decisionCount` = distinct entry IDs in the group.

**Upsert:** `ScoreType.CAPABILITY_DIMENSION`, `capabilityKey = capabilityTag`,
`dimensionKey = trustDimension`.

**Actor loop pass order:** capability → dimension → capability-dimension → global.
Each pass is independent and reads from `actorAttestations` already in memory.

---

## 5. `TrustGateService` — New Query Surface

Three new methods, mirroring the existing `dimensionScore` / `dimensionScores` /
`meetsThreshold` pattern:

```java
/**
 * Returns true if the actor's quality score for the given capability+dimension
 * meets or exceeds minScore. Returns false if no composite score has been computed.
 */
public boolean meetsQualityThreshold(String actorId, String capabilityTag,
        String dimension, double minScore);

/**
 * The actor's quality score for a specific capability+dimension, or empty if
 * not yet computed.
 */
public Optional<Double> qualityScore(String actorId, String capabilityTag, String dimension);

/**
 * All quality dimension scores for the given actor and capability, keyed by
 * dimension name. Empty map if none computed.
 */
public Map<String, Double> qualityScores(String actorId, String capabilityTag);
```

Existing methods (`currentScore`, `dimensionScore`, `dimensionScores`, `meetsThreshold`)
are updated internally to use the new typed repo methods — signatures and behaviour
unchanged.

---

## 6. Federation Export

### New record

```java
/** A capability-scoped quality dimension score for one actor. */
public record CapabilityDimensionScoreExport(
        String capabilityTag,
        String dimension,
        double score,
        int sampleCount) {
}
```

### `ActorExport`

```java
public record ActorExport(
        String actorId,
        ActorType actorType,
        GlobalScoreExport globalScore,
        List<CapabilityScoreExport> capabilityScores,
        List<DimensionScoreExport> dimensionScores,
        List<CapabilityDimensionScoreExport> capabilityDimensionScores) {
}
```

### `TrustExportService`

Populates `capabilityDimensionScores` in `exportAll`, `exportActor`, and `exportDelta`
by filtering `CAPABILITY_DIMENSION` rows — same pattern as the existing `DIMENSION`
row projection.

### `JpaTrustImportService`

Handles `capabilityDimensionScores` in the payload with seed-if-absent logic,
calling `upsert` with `ScoreType.CAPABILITY_DIMENSION` and the appropriate
`capabilityKey` / `dimensionKey`.

---

## 7. Testing

### Unit tests (no Quarkus runtime)

- `TrustScoreComputerTest` — `computeDimensionScore` already tested; verify it handles
  the attestation subsets the new pass will feed it
- `TrustGateServiceTest` — extend `StubRepository` with new typed methods; add tests
  for `qualityScore`, `qualityScores`, `meetsQualityThreshold`

### Integration tests (`@QuarkusTest`)

- `TrustScoreCapabilityDimensionIT` — end-to-end: persist attestations with both
  `capabilityTag` and `trustDimension`, run `TrustScoreJob`, assert `CAPABILITY_DIMENSION`
  rows written with correct scores
- Assert `CAPABILITY` and `DIMENSION` rows are unaffected by the new pass
- Assert attestations with only `capabilityTag` (no `trustDimension`) produce no
  `CAPABILITY_DIMENSION` rows
- `TrustGateServiceIT` — assert `meetsQualityThreshold`, `qualityScore`,
  `qualityScores` return correct values after job run

### Federation round-trip

- `TrustExportServiceIT` — assert `capabilityDimensionScores` populated in export payload
- `TrustImportServiceIT` — assert `JpaTrustImportService` seeds `CAPABILITY_DIMENSION`
  rows correctly from an imported payload

### Schema validation

- Existing `ActorTrustScoreRepositoryIT` covers the unique constraint; extend to
  verify the CHECK constraint rejects inconsistent rows (e.g. `CAPABILITY` with null
  `capability_key`)

---

## 8. Deferred / Out of Scope

- **#77** — grep downstream repos (`casehub-work`, `casehub-qhorus`, `casehub-engine`,
  `claudony`, `casehub-devtown`) for direct `scopeKey` references after #76 ships
- **#78** — ADR documenting decay-weighted average vs Bayesian Beta rationale
- **PLATFORM.md** — update capability ownership table with new `TrustGateService`
  methods after implementation
