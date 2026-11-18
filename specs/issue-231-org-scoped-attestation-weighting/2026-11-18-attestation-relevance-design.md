# Attestation Relevance — Org-Scoped Trust Weighting

**Issue:** casehubio/ledger#231
**Epic:** casehubio/casehub-ops#217 (Phase 2 — org-scoped trust and capacity)
**Depends on:** casehubio/eidos#212 (org proximity API) — landed

## Summary

Add a per-attestation relevance weighting layer to the trust computation pipeline. When an agent's trust score is computed, each attestation is weighted by the organisational proximity between the attestor and the scored actor at the time the attestation was written. Same-unit attestations receive full weight; cross-unit attestations receive a configurable discount. The system falls back to equal weighting when no org structure is available. The design introduces an extensible SPI (`AttestationRelevancePolicy`) in the ledger, with the org-proximity implementation shipped by eidos — preserving ledger's domain-agnostic boundary.

## Motivation

An attestation's value comes from two qualities in tension:

- **Accuracy** — same-unit peers share operational context, tools, and domain knowledge. Their judgments are more informed.
- **Independence** — same-unit peers also share biases, training data, and social incentives. Their judgments are correlated. Cross-unit attestations provide genuinely independent signals.

In neocortex multi-agent workflows, agents are organised into units with shared capabilities and prompts. A trust system that only privileges proximity creates echo chambers: a unit of agents can self-attest to high trust through correlated positive attestations. The design addresses this through mass-preserving normalisation (§7) — relevance re-mixes whose evidence dominates without reducing total evidence mass.

The existing weight chain (`decay × confidence × credibility`) includes no pair-scoped factor. Credibility (`AttestorCredibilityPolicy`) is attestor-scoped — "is Y a reliable judge in general?" Relevance is pair-scoped — "is Y's judgment pertinent to X specifically, for this attestation?" These are independent concepts requiring separate SPIs.

## Architecture

```
┌─────────────────────────────────────────────┐
│  api module                                 │
│  ├── AttestationRelevancePolicy (SPI)       │
│  ├── AttestationRelevanceFactor (SPI)       │
│  ├── AttestationRelevanceContext (record)    │
│  └── RelevanceAssessment (record)           │
├─────────────────────────────────────────────┤
│  ledger-core                                │
│  ├── NoOpAttestationRelevancePolicy         │
│  ├── FactorAggregatingRelevancePolicy       │
│  │   └── iterates Instance<Factor>, product │
│  │       with floor, normalises (§7)        │
│  └── AttestationWeights (parameter object)  │
├─────────────────────────────────────────────┤
│  runtime                                    │
│  └── CDI producers for relevance pipeline   │
├─────────────────────────────────────────────┤
│  eidos (separate repo)                      │
│  ├── OrgProximityRelevanceFactor            │
│  │   └── reads org_distance from context    │
│  │       maps via weight table (config)     │
│  └── OrgDistanceAttestationEnricher         │
│      └── write-time: orgDistance() → column  │
└─────────────────────────────────────────────┘
```

**Dependency direction:** eidos → ledger-api (existing). Ledger has zero eidos dependency. The org-proximity factor displaces the NoOp by classpath presence, following the `AttestorCredibilityPolicy` + `NoOpAttestorCredibilityPolicy` pattern.

## SPI Contract

All types live in `io.casehub.ledger.api.spi`.

### AttestationRelevanceContext

```java
/**
 * Per-attestation context for relevance assessment. Carries the attestation's
 * identity, capability, and write-time org snapshot — everything a relevance
 * factor needs to compute a weight without I/O.
 *
 * <p>This record can gain fields (nullable or with defaults) without breaking
 * existing implementations, enabling future factors (capability overlap,
 * collaboration recency, domain specialisation) to use the same SPI.
 */
public record AttestationRelevanceContext(
        UUID   attestationId,
        String attestorId,
        ActorType attestorType,
        String attestorRole,
        String scoredActorId,
        String tenancyId,
        String capabilityTag,
        String trustDimension,
        Integer orgDistance       // nullable — from write-time snapshot; null = no org structure
) {}
```

### RelevanceAssessment

```java
/**
 * The relevance weight for one attestation, with an optional diagnostic reason.
 *
 * <p>Weight is in [0.0, 1.0]. The compact constructor validates the range and
 * rejects NaN. The {@code reason} field is for diagnostics and logging
 * (e.g. {@code "org-distance:3, weight:0.65"}), not for persistence or
 * compliance reporting.
 */
public record RelevanceAssessment(double weight, String reason) {
    public static final RelevanceAssessment NEUTRAL = new RelevanceAssessment(1.0, null);

    public RelevanceAssessment {
        if (Double.isNaN(weight) || weight < 0.0 || weight > 1.0) {
            throw new IllegalArgumentException("weight must be in [0.0, 1.0], got: " + weight);
        }
    }
}
```

### AttestationRelevanceFactor

```java
/**
 * A single relevance factor contributed by any module on the classpath.
 *
 * <p>Each factor computes one dimension of attestation relevance (org proximity,
 * capability overlap, collaboration recency, etc.). The ledger aggregates all
 * discovered factors via {@code Instance<AttestationRelevanceFactor>}, computes
 * the product of their weights (with a global floor), and normalises per actor
 * to preserve evidence mass (see {@link AttestationRelevancePolicy}).
 *
 * <p>Implementations must be CDI beans ({@code @ApplicationScoped}).
 */
public interface AttestationRelevanceFactor {

    RelevanceAssessment assess(AttestationRelevanceContext ctx);

    default Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) {
        Map<UUID, RelevanceAssessment> result = new LinkedHashMap<>();
        for (AttestationRelevanceContext ctx : ctxs) {
            result.put(ctx.attestationId(), assess(ctx));
        }
        return result;
    }

    /** Short identifier for diagnostics, e.g. {@code "org-proximity"}. */
    String name();
}
```

### AttestationRelevancePolicy

```java
/**
 * Aggregation point for attestation relevance. The default implementation
 * iterates all {@link AttestationRelevanceFactor} beans, computes the product
 * of their weights (floored to prevent compounding from driving weights to
 * near-zero), and concatenates reasons.
 *
 * <p>This SPI sits alongside {@link AttestorCredibilityPolicy} in the trust
 * weight chain. Credibility is attestor-scoped (Y's general reliability).
 * Relevance is per-attestation and pair-scoped (how pertinent Y's judgment
 * is to X specifically, for this attestation, in this tenant).
 *
 * <p><b>Accuracy vs independence:</b> shared operational context makes
 * attestors' judgments more informed (accuracy), but also more correlated
 * (less independence). Implementations control this balance. The default
 * org-proximity factor is accuracy-biased: same-unit attestations get full
 * weight, cross-unit attestations get a configurable discount. Consumers
 * needing independence-biased weighting swap the factor or the policy.
 *
 * <p>With no factors on the classpath, all assessments are
 * {@link RelevanceAssessment#NEUTRAL} — backward-compatible.
 *
 * <p><b>Not retrieval relevance.</b> This is attestation-relevance for trust
 * scoring. Distinct from RAG retrieval relevance (neocortex) and semantic
 * routing relevance (engine-ai).
 */
public interface AttestationRelevancePolicy {

    RelevanceAssessment assess(AttestationRelevanceContext ctx);

    default Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) {
        Map<UUID, RelevanceAssessment> result = new LinkedHashMap<>();
        for (AttestationRelevanceContext ctx : ctxs) {
            result.put(ctx.attestationId(), assess(ctx));
        }
        return result;
    }
}
```

## Integration

### Weight Chain

The effective weight per attestation becomes:

```
effectiveWeight = decay × confidence × credibility × normalised_relevance
```

Where `normalised_relevance` is the raw relevance weight after mass-preserving normalisation (§7).

### AttestationWeights Parameter Object

```java
/**
 * Groups per-attestor credibility and per-attestation relevance lookups for
 * the trust computation pipeline. Prevents positional parameter errors when
 * both maps flow through TrustScoreComputer.
 */
public record AttestationWeights(
        Map<String, AttestorCredibilityPolicy.CredibilityAssessment> credibilityByAttestorId,
        Map<UUID, Double> relevanceByAttestationId
) {
    public static final AttestationWeights NEUTRAL = new AttestationWeights(Map.of(), Map.of());
}
```

Lives in `ledger-core` alongside `TrustScoreComputer`.

### TrustScoreComputer Changes

New overloads of `compute()` and `computeDimensionScore()` accept `AttestationWeights`:

```java
public ActorScore compute(List<LedgerEntry> decisions,
                          Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
                          Instant now,
                          AttestationWeights weights) { ... }
```

Inside the inner loop, after computing `rawWeight * credibilityWeight`:

```java
double relevanceWeight = weights.relevanceByAttestationId()
        .getOrDefault(attestation.id, 1.0);
double effectiveWeight = rawWeight * credibilityWeight * relevanceWeight;
```

Existing overloads delegate with `AttestationWeights.NEUTRAL` — backward-compatible.

### TrustScoreCalculator Changes

`computeAll()` gains a `scoredActorId` parameter:

```java
public ComputedScores computeAll(String scoredActorId,
                                 List<LedgerEntry> decisions,
                                 Map<UUID, List<LedgerAttestation>> attestationsByEntry,
                                 Instant now) { ... }
```

Inside `computeAll()`, after the existing credibility batch resolution:

1. Build `List<AttestationRelevanceContext>` from all attestations (using fields from `LedgerAttestation` plus `scoredActorId` and each entry's `tenancyId`).
2. Call `relevancePolicy.assessBatch(contexts)` → `Map<UUID, RelevanceAssessment>`.
3. Extract raw weights, apply mass-preserving normalisation (§7).
4. Build `AttestationWeights(credibilityMap, normalisedRelevanceMap)`.
5. Pass to `TrustScoreComputer`.

### PerActorTrustComputerCore

Already has `actorId`. Passes it as `scoredActorId` to `TrustScoreCalculator.computeAll()`.

## Write-Time Snapshot

### Schema

Add a nullable `org_distance` column to `ledger_attestation`:

```sql
ALTER TABLE ledger_attestation ADD COLUMN org_distance INTEGER;
```

Null means "no org structure available at write time" and maps to NEUTRAL (1.0) in the relevance policy. Existing attestations have null — no score disruption on upgrade.

### Eidos Enricher

Eidos ships `OrgDistanceAttestationEnricher` — a write-time hook (CDI observer on attestation persist) that:

1. Calls `orgAwareDispatch.orgDistance(attestorId, scoredActorId, tenancyId)`.
2. Maps -1 (unknown/disconnected) to null (not stored — same as "no org structure").
3. Sets `attestation.orgDistance` for values ≥ 0.

The enricher is absent when eidos is not on the classpath — the column stays null.

### Rationale

Follows ADR 0005 ("Group Membership Snapshot at WorkItem Creation"): exclusion must be deterministic, provider calls must not multiply latency, and the audit trail must explain the result. Org distance at compute time rewrites trust history after reorgs — a same-unit peer who moves units would retroactively lose proximity weight on all past attestations. The snapshot captures the relationship that actually existed when the attestation was made.

The snapshot also ensures consistency across all three `TrustScoreSource` strategies (Materialized, Cached, Computed) — no divergence after org changes without attestation changes.

## Mass-Preserving Normalisation

Relevance changes **whose** evidence dominates, never **how much** evidence exists. Without normalisation, a uniform 0.7 discount on all attestations reduces total evidence mass, pulling Beta scores toward the 0.5 prior — a 5.5% drop from topology alone for 3 SOUND attestations.

### Formula

After batch-resolving raw relevance weights `r_i` for all attestations in a trust computation:

```
normalisation_factor = Σ(raw_i × cred_i) / Σ(raw_i × cred_i × r_i)
normalised_relevance_i = r_i × normalisation_factor
```

Where `raw_i = decay_i × confidence_i` and `cred_i` is the credibility weight. The normalised weights satisfy `Σ(raw_i × cred_i × normalised_relevance_i) = Σ(raw_i × cred_i)`.

### Per-Pass Application

Normalisation applies uniformly to all four passes (capability Beta, global Beta, dimension weighted-average, capability×dimension). This gives consistent semantics: relevance is always a reweighting, never an evidence discount.

The existing `credibilityRetention` metric remains solely about credibility. A separate diagnostic metric (`relevanceEffect`) can be added if needed.

## Default Implementation (eidos)

### OrgProximityRelevanceFactor

`@ApplicationScoped` bean implementing `AttestationRelevanceFactor`. Name: `"org-proximity"`.

Reads `orgDistance` from the `AttestationRelevanceContext` (write-time snapshot) and maps it via a **distance-indexed weight table**:

| Index | Distance meaning | Default weight |
|-------|-----------------|---------------|
| 0 | Same agent | 1.0 |
| 1 | Same unit | 1.0 |
| 2 | Sibling units | 0.7 |
| 3 | 3 hops | 0.65 |
| ≥ 4 | Distant (last entry repeats) | 0.6 |

Special cases:
- `null` orgDistance → NEUTRAL (1.0). No org structure — equal weighting.
- -1 orgDistance → NEUTRAL (1.0). Unknown/disconnected — not penalised.

### Eidos Config

```properties
# Distance-indexed weight table (last entry repeats for all greater distances)
casehub.eidos.trust.relevance.weights=1.0,1.0,0.7,0.65,0.6
```

Validated at startup: non-empty, all values in [0.0, 1.0]. Spring parity via `@ConfigurationProperties`.

## Schema

Migration in `ledger-jpa-common/src/main/resources/db/ledger/migration/`:

The `org_distance` column is added to the `ledger_attestation` table definition in `V1000__ledger_initial_schema.sql` (no existing installations — all schema changes go into base migrations per project convention):

```sql
org_distance     INTEGER,          -- write-time org proximity snapshot; null = no org structure
```

Added after the `occurred_at` column in the `CREATE TABLE ledger_attestation` statement.

## Explicit Exclusions

**EigenTrust:** Relevance weighting is not applied to `EigenTrustComputer` edge weights. EigenTrust operates on its own local-trust matrix where self-attestation is already excluded. Applying relevance to edge weights would change convergence semantics and is deferred to a future issue.

**Consumer-relative trust:** Trust is actor-relative (how trustworthy is X?), not observer-relative (how trustworthy does X look from unit U's perspective?). `TrustScoreSource` is keyed by `actorId` alone. Personalised trust (personalised-PageRank/EigenTrust) would need a new read API and cannot be materialised. This is a defensible scope boundary — qhorus#501 and later epic phases may revisit.

## Testing Strategy

### Unit Tests (pure Java, no CDI)

- `RelevanceAssessmentTest` — compact constructor validates [0.0, 1.0], rejects NaN.
- `AttestationRelevanceContextTest` — record construction with nullable orgDistance.
- `TrustScoreComputerRelevanceTest` — verify the four-factor weight chain produces correct alpha/beta with known relevance weights. Test mass-preserving normalisation: uniform relevance must not change scores vs no relevance. Test mixed relevance: same-unit attestations weighted higher than cross-unit.
- `TrustScoreCalculatorRelevanceTest` — verify `scoredActorId` flows through, batch resolution calls the policy, normalisation is applied.
- `FactorAggregatingRelevancePolicyTest` — verify product-with-floor, reason concatenation, NEUTRAL with no factors.

### Integration Tests

- `OrgProximityRelevanceFactorTest` (in eidos) — weight table lookup, null/unknown → NEUTRAL, distance beyond table → last entry.
- `OrgDistanceAttestationEnricherTest` (in eidos) — write-time snapshot persists org_distance; -1 stored as null.
- `TrustScoreSourceContractTest` — extend to verify all three sources agree with relevance active.

## Cross-Repo Implementation

### Ledger (casehubio/ledger)

- **api:** `AttestationRelevancePolicy`, `AttestationRelevanceFactor`, `AttestationRelevanceContext`, `RelevanceAssessment`
- **ledger-core:** `NoOpAttestationRelevancePolicy`, `FactorAggregatingRelevancePolicy`, `AttestationWeights`, normalisation logic in `TrustScoreCalculator`
- **ledger-core:** `TrustScoreComputer` new overloads with `AttestationWeights`
- **ledger-jpa-common:** `org_distance` column on `LedgerAttestation`, schema migration
- **runtime:** CDI producers for the relevance pipeline
- **persistence-memory:** `InMemoryLedgerEntryRepository` updated for new attestation field

### Eidos (casehubio/eidos)

- **org-runtime:** `OrgProximityRelevanceFactor`, `OrgDistanceAttestationEnricher`, config mapping
- Tests for the factor and enricher

## References

- `api/src/main/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicy.java` — SPI pattern precedent
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java` — weight chain
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreCalculator.java` — batch resolution pattern
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/NoOpAttestorCredibilityPolicy.java` — NoOp default pattern
- `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java` — attestation fields
- `eidos/api/src/main/java/io/casehub/eidos/api/OrgAwareDispatch.java` — org distance API
- `eidos/eidos-core/src/main/java/io/casehub/eidos/core/org/NoOpOrgAwareDispatch.java` — NoOp fallback
- ADR 0005 — Group Membership Snapshot at WorkItem Creation (write-time snapshot precedent)
- ADR 0005 — LedgerEntryEnricher pipeline (`Instance<>` pipeline precedent)
- qhorus#505 — write-time org resolution precedent
- casehubio/casehub-ops#217 — parent epic (Phase 2 — org-scoped trust and capacity)
- casehubio/eidos#212 — org proximity API (dependency, landed)
- `parent/docs/platform/dependency-map.md` — build order (ledger has no casehubio deps)
