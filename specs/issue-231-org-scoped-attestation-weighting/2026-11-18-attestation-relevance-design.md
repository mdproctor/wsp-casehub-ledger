# Attestation Relevance — Org-Scoped Trust Weighting

**Issue:** casehubio/ledger#231
**Epic:** casehubio/casehub-ops#217 (Phase 2 — org-scoped trust and capacity)
**Depends on:** casehubio/eidos#212 (org proximity API) — landed

## Summary

Add a per-attestation relevance weighting layer to the trust computation pipeline. When an agent's trust score is computed, each attestation is re-weighted by the organisational proximity between the attestor and the scored actor at the time the attestation was written. Relative to cross-unit attestors, same-unit attestors' judgments carry more influence. The total evidence mass per pass is preserved — relevance shifts whose evidence dominates, never how much evidence exists. The system falls back to equal weighting when no org structure is available. The design introduces an extensible factor SPI (`AttestationRelevanceFactor`) in the ledger, with the org-proximity implementation shipped by eidos — preserving ledger's domain-agnostic boundary.

## Motivation

An attestation's value comes from two qualities in tension:

- **Accuracy** — same-unit peers share operational context, tools, and domain knowledge. Their judgments are more informed.
- **Independence** — same-unit peers also share biases, training data, and social incentives. Their judgments are correlated. Cross-unit attestations provide genuinely independent signals.

In neocortex multi-agent workflows, agents are organised into units with shared capabilities and prompts. The default org-proximity factor is **accuracy-biased**: it increases in-unit attestation influence relative to cross-unit attestations. This is the direction ledger#231 and the epic ask for — "shared operational context makes a peer's judgment more relevant."

Mass-preserving normalisation (§7) prevents topology from shrinking total evidence — without it, any uniform discount pulls Beta scores toward the 0.5 prior. However, normalisation does **not** prevent in-unit amplification. By design, same-unit attestations receive proportionally more influence. This widens the "anti-gaming gap" noted in ARC42 §Active Risks: a unit of agents whose positive attestations correlate will produce higher trust scores than the same performance attested by cross-unit peers. An independence-aware factor (e.g. a cap on same-unit share of a group's mass) can be contributed through the factor pipeline as a future mitigation (see §Follow-Up Issues).

The existing weight chain (`decay × confidence × credibility`) includes no pair-scoped factor. Credibility (`AttestorCredibilityPolicy`) is attestor-scoped — "is Y a reliable judge in general?" Relevance is pair-scoped — "is Y's judgment pertinent to X specifically, for this attestation?" These are independent concepts requiring separate SPIs.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  api module                                     │
│  ├── AttestationRelevanceFactor (SPI)           │
│  ├── LedgerAttestationEnricher (SPI)            │
│  ├── AttestationRelevanceContext (record)        │
│  └── RelevanceAssessment (record)               │
├─────────────────────────────────────────────────┤
│  ledger-core                                    │
│  ├── RelevanceComposer (concrete)               │
│  │   └── iterates Instance<Factor>, product     │
│  │       with floor, mean-fills unknowns        │
│  └── AttestationWeights (parameter object)      │
├─────────────────────────────────────────────────┤
│  runtime / ledger-spring                        │
│  └── Producers/config for relevance pipeline    │
│  └── Attestation enricher pipeline in repos     │
├─────────────────────────────────────────────────┤
│  eidos (separate repo)                          │
│  ├── OrgProximityRelevanceFactor                │
│  │   └── reads "org-distance" from snapshot     │
│  │       maps via weight table (config)         │
│  └── OrgDistanceAttestationEnricher             │
│      └── write-time: orgDistance() → snapshot    │
└─────────────────────────────────────────────────┘
```

**Dependency direction:** eidos → ledger-api (existing). Ledger has zero eidos dependency. The org-proximity factor activates by classpath presence — with zero factors, `RelevanceComposer` returns NEUTRAL for all assessments.

## SPI Contract

### AttestationRelevanceContext

Lives in `io.casehub.ledger.api.spi`.

```java
/**
 * Per-attestation context for relevance assessment. Carries the attestation's
 * identity, capability, timing, and write-time snapshot — everything a
 * relevance factor needs to compute a weight without I/O.
 *
 * <p>The {@code snapshot} map is the extensibility mechanism: each enricher
 * writes its own keys (e.g. {@code "org-distance": 2}), and each factor reads
 * its own key. Adding a new factor requires only a new enricher and factor
 * bean — no record or schema changes in ledger.
 *
 * <p>Adding a component to this record is a source-breaking change for callers
 * (canonical constructor changes). The snapshot map avoids this for
 * enricher-provided data.
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
        Instant occurredAt,
        Map<String, Object> snapshot  // nullable — write-time enricher data; null = no enrichers ran
) {}
```

### RelevanceAssessment

Lives in `io.casehub.ledger.api.spi`.

```java
/**
 * The raw relevance weight for one attestation, with an optional diagnostic reason.
 *
 * <p>Weight is in [0.0, 1.0]. After mass-preserving normalisation (§7),
 * the effective weight used in computation may exceed 1.0 for above-mean
 * attestations — that normalisation happens in {@code TrustScoreComputer},
 * not here.
 *
 * <p>The {@code reason} field is for diagnostics and DEBUG logging
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

Lives in `io.casehub.ledger.api.spi`.

```java
/**
 * A single relevance factor contributed by any module on the classpath.
 *
 * <p>Each factor computes one dimension of attestation relevance (org proximity,
 * capability overlap, collaboration recency, etc.). The ledger's
 * {@code RelevanceComposer} aggregates all discovered factors, computes the
 * product of their weights (floored at a configurable minimum to prevent
 * compounding from driving weights to near-zero), and feeds the raw weights
 * into per-pass mass-preserving normalisation inside {@code TrustScoreComputer}.
 *
 * <p>With no factor beans on the classpath, all assessments are
 * {@link RelevanceAssessment#NEUTRAL} — backward-compatible.
 *
 * <p><b>Accuracy vs independence:</b> shared operational context makes
 * attestors' judgments more informed (accuracy), but also more correlated
 * (less independence). The default org-proximity factor is accuracy-biased.
 * An independence-biased factor can be contributed through this same SPI.
 *
 * <p><b>Not retrieval relevance.</b> This is attestation-relevance for trust
 * scoring. Distinct from RAG retrieval relevance (neocortex) and semantic
 * routing relevance (engine-ai).
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

### LedgerAttestationEnricher

Lives in `io.casehub.ledger.api.spi`.

```java
/**
 * Write-time enricher for attestations — the sibling of ADR 0005's
 * {@code LedgerEntryEnricher} for entries.
 *
 * <p>Discovered via {@code Instance<LedgerAttestationEnricher>} (Quarkus) or
 * {@code List<LedgerAttestationEnricher>} (Spring). All three repositories
 * (JPA, Spring JPA, in-memory) invoke the pipeline after entry lookup and
 * tenant validation, before tokenisation and persist.
 *
 * <p>Enrichers populate the {@code relevanceSnapshot} JSON on the attestation.
 * Each enricher writes its own keys. The repository clears
 * {@code relevanceSnapshot} to null before running the pipeline, so
 * caller-supplied values are discarded — the snapshot is system-derived only.
 *
 * <p>Enrichers receive the raw (pre-tokenised) attestor identity. This is
 * critical: tokenised IDs are opaque and would return -1 from
 * {@code OrgAwareDispatch.orgDistance()}.
 *
 * <p>Failures are non-fatal: a failed enricher leaves its snapshot keys absent
 * and logs a warning. The attestation persists with whatever keys succeeded.
 */
public interface LedgerAttestationEnricher {

    void enrich(LedgerAttestation attestation, LedgerEntry entry);

    /** Short identifier for diagnostics. */
    String name();
}
```

## Integration

### Weight Chain

The effective weight per attestation becomes:

```
effectiveWeight = decay × confidence × credibility × normalised_relevance
```

Where `normalised_relevance` is the raw relevance weight after per-pass mass-preserving normalisation (§7).

### Self-Attestation Exclusion

`TrustScoreComputer.compute()` excludes attestations where `attestorId == entry.actorId` from the Beta computation. This matches `EigenTrustComputer`'s existing self-attestation exclusion. Self-attestation is the least independent signal and must not be boosted by proximity weighting.

### AttestationWeights Parameter Object

```java
/**
 * Groups per-attestor credibility and per-attestation raw relevance weights
 * for the trust computation pipeline. Prevents positional parameter errors.
 *
 * <p>{@code relevanceByAttestationId} contains <b>raw</b> (un-normalised)
 * weights from the {@code RelevanceComposer}. Normalisation happens per pass
 * inside {@code TrustScoreComputer.compute()}.
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

Replace the existing overload chain with a single canonical signature for each method:

```java
public ActorScore compute(List<LedgerEntry> decisions,
                          Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
                          Instant now,
                          AttestationWeights weights) { ... }

public OptionalDouble computeDimensionScore(
                          List<LedgerAttestation> dimensionAttestations,
                          Instant now,
                          AttestationWeights weights) { ... }
```

Tests pass `AttestationWeights.NEUTRAL`. The old overloads are removed — per the project's "breaking changes cost nothing" pre-release stance, there is no compatibility shim.

Inside `compute()`, normalisation happens per invocation (see §7). Inside `computeDimensionScore()`, relevance weights are applied directly (uniform scaling cancels in a weighted average, so normalisation is a no-op for dimension scores — relative weights are what matter).

`credibilityRetention` uses a **separate credibility-only accumulator** so it is not affected by relevance weights.

### TrustScoreCalculator Changes

`computeAll()` does NOT gain a `scoredActorId` parameter. Each attestation's context is built from the entry it belongs to: `entry.actorId` is the scored actor (retrieved via the `ledgerEntryId → decisions` mapping already present), and `entry.tenancyId` provides the tenancy. This avoids a redundant parameter that could disagree with the entry data.

Inside `computeAll()`, after the existing credibility batch resolution:

1. Build `List<AttestationRelevanceContext>` from all attestations, using fields from `LedgerAttestation`, the entry's `actorId` and `tenancyId`, and the attestation's `relevanceSnapshot`.
2. Call `relevanceComposer.assess(contexts)` → `Map<UUID, Double>` (raw weights).
3. Build `AttestationWeights(credibilityMap, rawRelevanceMap)`.
4. Pass to `TrustScoreComputer` — normalisation happens per pass inside `compute()`.

### RelevanceComposer (ledger-core)

Concrete class (not an SPI). Iterates `Instance<AttestationRelevanceFactor>`, computes the product of weights (floored at `casehub.ledger.trust-score.relevance.min-weight`, default 0.5, validated at startup in (0, 1]), and concatenates reasons. With zero factors, returns NEUTRAL for all assessments.

For attestations where all factors return NEUTRAL (e.g. no snapshot data), the raw weight is 1.0. Under normalisation, these attestations receive the **mean** relevance weight of the known attestations in the same pass group, so they are truly mass-neutral — neither boosted nor penalised relative to assessed peers. If a pass group has no known distances, relevance is a no-op for that group.

## Write-Time Snapshot

### Schema

Add a nullable `relevance_snapshot` TEXT column to `ledger_attestation`, added to the `CREATE TABLE` statement in `V1000__ledger_base_schema.sql` (no existing installations — all schema changes go into base migrations per project convention):

```sql
relevance_snapshot  TEXT,  -- write-time enricher JSON snapshot; null = no enrichers ran
```

The column holds JSON in the same pattern as `supplementJson` on entries. Each attestation enricher writes its own keys: e.g. `{"org-distance": 2}`. Adding a future factor that needs a write-time snapshot requires only a new enricher and factor bean — no ledger schema change.

Both ORM mapping files need the column:
- `runtime/src/main/resources/META-INF/orm.xml`
- `ledger-spring-jpa/src/main/resources/META-INF/orm.xml`

The field is added to `io.casehub.ledger.api.model.LedgerAttestation` (api module, where all attestation fields live).

Null means "no enrichers ran" (eidos not on classpath, or attestation predates the feature). Under normalisation, null-snapshot attestations receive the mean relevance of the group (§7).

### Attestation Enricher Pipeline

New SPI: `LedgerAttestationEnricher` in `api` (see §SPI Contract above). The pipeline runs in all three repository implementations:

1. `JpaLedgerEntryRepository.saveAttestation()` — Quarkus
2. `SpringJpaLedgerEntryRepository.saveAttestation()` — Spring
3. `InMemoryLedgerEntryRepository.saveAttestation()` — testing

**Pipeline order** (in each repository's `saveAttestation`):
1. Look up the parent entry, validate tenant
2. **Clear** `attestation.relevanceSnapshot` to null (prevents caller-supplied values — R1-06)
3. Run `LedgerAttestationEnricher` pipeline on the **raw** (pre-tokenised) attestor identity
4. Tokenise attestor identity (`actorIdentityProvider.tokenise`)
5. Persist

Failures in enrichers are non-fatal (leave their snapshot keys absent, log a warning) — matching ADR 0005's entry enricher behaviour. Needs an ADR update or new ADR to cover attestation enrichment.

### Eidos Enricher

Eidos ships `OrgDistanceAttestationEnricher implements LedgerAttestationEnricher` in a module that depends on `casehub-ledger-api` (a dependency edge that must be added to eidos `org-runtime` and reflected in `parent/docs/platform/dependency-map.md`).

The enricher:
1. Calls `orgAwareDispatch.orgDistance(attestorId, scoredActorId, tenancyId)` using raw (pre-tokenised) identities.
2. Maps ≥ 0 results to `{"org-distance": N}` in the snapshot JSON.
3. Maps -1 (unknown/disconnected) to no entry (same as null — "no org structure for this pair").

The enricher is absent when eidos is not on the classpath — the snapshot stays null.

### Rationale

Follows ADR 0005 ("Group Membership Snapshot at WorkItem Creation"): exclusion must be deterministic, provider calls must not multiply latency, and the audit trail must explain the result. Org distance at compute time rewrites trust history after reorgs — a same-unit peer who moves units would retroactively lose proximity weight on all past attestations. The snapshot captures the relationship that actually existed when the attestation was made.

The snapshot also ensures consistency across all three `TrustScoreSource` strategies (Materialized, Cached, Computed) — no divergence after org changes without attestation changes.

## Mass-Preserving Normalisation

Relevance changes **whose** evidence dominates within a pass, never **how much** evidence exists. Without normalisation, any uniform discount on attestations reduces total evidence mass, pulling Beta scores toward the 0.5 prior.

### Where Normalisation Happens

Normalisation runs inside `TrustScoreComputer.compute()`, **per invocation** — over exactly the attestations of that capability/global group. It does **not** run in `computeDimensionScore()`: the dimension pass is a normalised weighted average (`Σwx/Σw`), where multiplying every weight by the same scalar has no effect. Only relative weights inside each group matter for dimensions.

### Algorithm (inside `compute()`)

Two passes over the attestation group:

**Pass 1 — compute normalisation factor:**
```
sum_base = 0, sum_relevance = 0
for each attestation:
    rawWeight = decay × confidence
    credWeight = credibilityByAttestorId.getOrDefault(attestorId, 1.0)
    relevWeight = relevanceByAttestationId.getOrDefault(attestationId, 1.0)
    sum_base += rawWeight × credWeight
    sum_relevance += rawWeight × credWeight × relevWeight
if sum_relevance == 0: normFactor = 1.0  // guard: no reweighting
else: normFactor = sum_base / sum_relevance
```

**Pass 2 — accumulate alpha/beta with normalised weights:**
```
for each attestation:
    rawWeight = decay × confidence
    credWeight = ...
    relevWeight = relevanceByAttestationId.getOrDefault(attestationId, 1.0)
    normalisedRelevance = relevWeight × normFactor  // may exceed 1.0 — do NOT clamp
    effectiveWeight = rawWeight × credWeight × normalisedRelevance
    // accumulate alpha/beta as today
    // credibilityRetention uses rawWeight × credWeight only (separate accumulator)
```

### Unknown/Null Attestations Under Normalisation

Attestations with no snapshot data (null or missing factor keys) receive raw relevance 1.0 from `RelevanceComposer`. Under normalisation, the effective question is: does 1.0 represent "same as same-unit" or "unknown — be neutral"?

To be truly neutral, unknown attestations are assigned the **mean raw relevance** of the known attestations in the same pass group before normalisation. If a group has no known distances, all relevance is 1.0 and normalisation is a no-op. This prevents legacy attestations (pre-feature) or unstructured attestors from outranking known cross-unit peers simply because they default to the maximum raw weight.

### Testable Invariant

**If every attestation in a pass has equal relevance, the score is unchanged vs no relevance.** This holds for both Beta and dimension passes.

## Default Implementation (eidos)

### OrgProximityRelevanceFactor

`@ApplicationScoped` bean implementing `AttestationRelevanceFactor`. Name: `"org-proximity"`.

Reads `"org-distance"` from the `AttestationRelevanceContext.snapshot()` and maps it via a **distance-indexed weight table**:

| Index | Eidos LCA hop distance meaning | Default weight |
|-------|-------------------------------|---------------|
| 1 | Same unit, or direct parent/child unit | 1.0 |
| 2 | Sibling units, or grandparent/grandchild | 0.7 |
| 3 | 3 LCA hops | 0.65 |
| ≥ 4 | Distant (last entry repeats) | 0.6 |

**Note on eidos LCA semantics:** `DefaultOrgAwareDispatch.orgDistance()` returns `max(1, i+j)` where `i` and `j` are ancestor-chain positions. Distance 1 covers both same-unit and direct parent/child. Distance 0 means same agent (self-attestation), which is excluded from Beta passes (§Integration).

Special cases:
- No `"org-distance"` key in snapshot → NEUTRAL (1.0). No org structure — equal weighting.
- `"org-distance"` value -1 → NEUTRAL (1.0). Unknown/disconnected — not penalised.

### Eidos Config

```properties
# Distance-indexed weight table (last entry repeats for all greater distances)
# Index 0 is never used (self-attestation excluded); index 1 = same/parent-child unit
casehub.eidos.trust.relevance.weights=1.0,0.7,0.65,0.6
```

Validated at startup: non-empty, all values in [0.0, 1.0].

### Ledger Composition Config

```properties
# Floor for the product of all relevance factors (prevents compounding to near-zero)
# Effectively caps the maximum influence ratio between any two attestations at 1/min-weight
casehub.ledger.trust-score.relevance.min-weight=0.5
```

Validated at startup: must be in (0.0, 1.0]. This is the only relevance config owned by ledger. It governs the composition pipeline, not any specific factor. Requires entries in both `LedgerConfig` (`@ConfigMapping`) and `LedgerConfigurationProperties` (Spring).

## Explicit Exclusions

**EigenTrust:** Relevance weighting is not applied to `EigenTrustComputer` edge weights. EigenTrust operates on its own local-trust matrix where self-attestation is already excluded. Applying relevance to edge weights would change convergence semantics and is deferred to a follow-up issue.

**Consumer-relative trust:** Trust is actor-relative (how trustworthy is X?), not observer-relative (how trustworthy does X look from unit U's perspective?). `TrustScoreSource` is keyed by `actorId` alone. Personalised trust (personalised-PageRank/EigenTrust) would need a new read API and cannot be materialised. This is a defensible scope boundary — rejected because the current materialisation model cannot support it.

## Testing Strategy

### Unit Tests (pure Java, no CDI)

- `RelevanceAssessmentTest` — compact constructor validates [0.0, 1.0], rejects NaN.
- `AttestationRelevanceContextTest` — record construction with nullable snapshot.
- `TrustScoreComputerRelevanceTest`:
  - Four-factor weight chain produces correct alpha/beta with known relevance weights.
  - **Per-pass invariance:** every attestation in one capability group at equal relevance gives unchanged score, even when other groups differ.
  - **Zero-mass guard:** all credibility 0, or all confidence 0, gives 0.5 and no NaN.
  - **Self-attestation excluded** from Beta accumulation.
  - `credibilityRetention` unchanged by relevance weights (separate accumulator).
  - Mixed null/known groups: null-snapshot attestations receive mean relevance of known peers.
- `TrustScoreCalculatorRelevanceTest` — batch resolution calls the composer, context built from entry data, `AttestationWeights` passed to computer.
- `RelevanceComposerTest` — product-with-floor, reason concatenation, NEUTRAL with no factors, floor validation rejects ≤ 0 and > 1.

### Integration Tests

- `OrgProximityRelevanceFactorTest` (in eidos) — weight table lookup, missing/unknown → NEUTRAL, distance beyond table → last entry, hierarchical distance cases (parent/child, grandparent, aunt/niece, multi-unit).
- `OrgDistanceAttestationEnricherTest` (in eidos) — write-time snapshot persists to `relevance_snapshot` JSON; -1 stored as absent key; runs on raw pre-tokenised identity.
- `LedgerAttestationEnricherPipelineTest` — enricher invoked before tokenisation; caller-supplied snapshot cleared; failure leaves null; `relevance_snapshot` round-trips through JPA, Spring JPA, and in-memory.
- `TrustScoreSourceContractTest` — extend to verify all three sources agree with relevance active. Thread the new `RelevanceComposer` argument through the two construction sites.
- **Spring context wiring** — zero and one factor bean.

## Cross-Repo Implementation

### Ledger (casehubio/ledger)

- **api:** `AttestationRelevanceFactor`, `LedgerAttestationEnricher`, `AttestationRelevanceContext`, `RelevanceAssessment`; `relevanceSnapshot` field on `LedgerAttestation`
- **ledger-core:** `RelevanceComposer`, `AttestationWeights`; `TrustScoreComputer` revised to single `AttestationWeights` signature + per-pass normalisation + self-attestation exclusion; `TrustScoreCalculator` builds context from entry data
- **runtime:** CDI producers for `RelevanceComposer` and attestation enricher pipeline; `LedgerConfig` gains `trust-score.relevance.min-weight`
- **ledger-spring:** `LedgerConfigurationProperties` gains min-weight; `LedgerManualConfig`/`LedgerTrustConfig` wire `RelevanceComposer` and enricher list
- **ledger-jpa-common:** `relevance_snapshot TEXT` column in V1000; both `orm.xml` files
- **persistence-memory:** enricher pipeline invocation in `InMemoryLedgerEntryRepository.saveAttestation()`

### Eidos (casehubio/eidos)

- **org-runtime** (needs new `casehub-ledger-api` dependency edge): `OrgProximityRelevanceFactor`, `OrgDistanceAttestationEnricher`, config mapping
- Tests for the factor and enricher
- Update `parent/docs/platform/dependency-map.md` for the new edge

## Follow-Up Issues

The following items are out of scope for #231 and need GitHub issues filed:

1. **Eidos companion issue** — `OrgProximityRelevanceFactor` and `OrgDistanceAttestationEnricher` implementation in eidos. ledger#231 cannot close while the eidos half is untracked.
2. **EigenTrust relevance weighting** — apply relevance to EigenTrust edge weights (deferred from §Explicit Exclusions).
3. **Independence-aware / anti-collusion factor** — cap on same-unit share of a group's mass, or independence-biased weight table, contributed through the factor pipeline.
4. **Self-attestation exclusion on Beta path** — if not completed within #231, file separately. The spec requires it; it must not be deferred without tracking.
5. **ADR for AttestationRelevanceFactor SPI** — follows precedent of ADR 0007 (DecayFunction), ADR 0008 (GlobalScoreStrategy), ADR 0009 (continuous scores).
6. **ARC42 updates** — §Active Risks (in-unit amplification trade-off), L-layer trust section, relevance in the weight chain.

## References

- `api/src/main/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicy.java` — SPI pattern precedent
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java` — weight chain, self-attestation handling
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreCalculator.java` — batch resolution pattern, per-pass grouping
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/NoOpAttestorCredibilityPolicy.java` — NoOp default pattern
- `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java` — attestation fields, relevanceSnapshot location
- `eidos/api/src/main/java/io/casehub/eidos/api/OrgAwareDispatch.java` — org distance API, LCA hop semantics
- `eidos/org-runtime/src/main/java/.../DefaultOrgAwareDispatch.java` — `max(1, i+j)` distance formula
- `eidos/eidos-core/src/main/java/io/casehub/eidos/core/org/NoOpOrgAwareDispatch.java` — always returns -1
- ADR 0005 — Group Membership Snapshot at WorkItem Creation (write-time snapshot precedent)
- ADR 0005 — LedgerEntryEnricher pipeline (`Instance<>` pipeline precedent, non-fatal failure)
- qhorus#505 — write-time org resolution precedent
- casehubio/casehub-ops#217 — parent epic (Phase 2 — org-scoped trust and capacity)
- casehubio/eidos#212 — org proximity API (dependency, landed)
- `parent/docs/platform/dependency-map.md` — build order (ledger has no casehubio deps)
- ARC42STORIES.MD §Active Risks — anti-gaming gap
- `runtime/src/main/resources/META-INF/orm.xml` — ORM mapping (needs relevance_snapshot)
- `ledger-spring-jpa/src/main/resources/META-INF/orm.xml` — ORM mapping (needs relevance_snapshot)
