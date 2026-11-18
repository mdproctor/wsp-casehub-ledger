# Decisions — #231 Org-Scoped Attestation Weighting

## D1: SPI abstraction level — per-attestation context record, not bare pair signature

**Choice:** New SPI `AttestationRelevancePolicy` with `assess(AttestationRelevanceContext ctx)` signature. The context record carries: `attestationId`, `attestorId`, `attestorType`, `attestorRole`, `scoredActorId`, `tenancyId`, `capabilityTag`, `trustDimension`, `orgDistance` (nullable Integer, from write-time snapshot — see D9). The record can gain fields without breaking implementations. Batch variant: `assessBatch(List<AttestationRelevanceContext> contexts)` returning `Map<UUID, RelevanceAssessment>` keyed by attestation ID. Lives in `api` module.
**Alternatives:**
- Bare pair signature `assess(attestorId, scoredActorId, tenancyId)` — insufficient for future factors. Domain specialisation needs `capabilityTag`/`trustDimension`. Collaboration recency needs `occurredAt`. Write-time org context needs the snapshot field. Each extension would force a signature change, defeating the "without changing the interface" goal.
- Fold into `AttestorCredibilityPolicy` — credibility is attestor-scoped (`assess(attestorId)`) with no context about who's being scored. Relevance is per-attestation and pair-scoped. Fundamental concept mismatch.
- Hop-count-only function `weight(int orgDistance)` — too narrow for any future factor. No compositional structure.
**Rationale:** Credibility asks "is Y a reliable judge in general?" Relevance asks "is Y's judgment pertinent to X specifically, for this attestation?" These are independent concepts. The context record is the only form that actually delivers extensibility without interface changes — adding a field to the record is backward-compatible (new fields have defaults or are nullable). Per-attestation keying handles cross-tenant correctly (same attestor can attest in different tenants at different org distances).
**Trade-offs:** Heavier SPI surface than a bare double or pair signature. Justified: each field in the context record corresponds to a concrete future factor identified during analysis (org proximity, capability overlap, domain specialisation, collaboration recency).
**Sources:** TrustScoreComputer.java (weight chain), AttestorCredibilityPolicy.java (SPI pattern), eidos OrgAwareDispatch.java, LedgerAttestation.java (fields available at call site), ADR 0005 (write-time snapshot precedent)
**Exploration:** deep-analysis
**Status:** revised (was: captured) — R1-05, R1-03

## D2: Naming — `AttestationRelevancePolicy` (per-attestation scope)

**Choice:** Name the SPI `AttestationRelevancePolicy`. The "Attestation" prefix is accurate because the SPI operates on per-attestation context (D1's context record), not on attestor identity alone.
**Alternatives:**
- `AttestorRelevancePolicy` — would be correct if the signature were attestor-scoped (like `AttestorCredibilityPolicy`). With the per-attestation context record, "Attestor" is misleading.
- `ProximityWeightFunction` — names the first implementation, not the concept. Would become misleading when capability overlap or collaboration recency implementations arrive.
- `AttestationContextPolicy` — too vague; "context" doesn't convey the weight-modifying role.
**Rationale:** Name follows the signature scope. The sibling SPI `AttestorCredibilityPolicy` is attestor-scoped and named accordingly. This SPI is per-attestation-scoped and named accordingly. Javadoc distinguishes this from retrieval relevance (neocortex RAG) and routing relevance (engine-ai).
**Trade-offs:** None significant. The naming convention is consistent with the existing SPI.
**Sources:** AttestorCredibilityPolicy.java (naming convention), D1 (signature scope)
**Exploration:** quick
**Status:** revised (was: captured) — R1-15

## D3: Return type — `RelevanceAssessment(weight, reason)` record with validated range

**Choice:** Return a `RelevanceAssessment` record with `weight` (double, validated [0.0, 1.0] in compact constructor) and `reason` (nullable String), plus static `NEUTRAL` constant (weight 1.0, null reason).
**Alternatives:**
- Bare `double` — simpler, but no diagnostics. When debugging why an actor's trust score is lower than expected, the reason string immediately shows "org-distance:4, weight:0.65" vs requiring a separate debug session.
- Full record with flags (like CredibilityAssessment) — no concrete flag use case yet. Add flags when needed; the record is extensible.
**Rationale:** The `reason` field is for diagnostics and logging, not for compliance reporting or persistence. (Note: `CredibilityAssessment.reason` has zero production usages currently — but the diagnostic path exists for when it's needed.) The compact constructor validates weight ∈ [0.0, 1.0] and rejects NaN/negative/above-1 — relevance cannot exceed 1.0 (see D10 for mass-preserving normalisation, which operates at the calculator level). Following the `CredibilityAssessment` pattern for consistency, minus flags (YAGNI).
**Trade-offs:** Record allocation vs bare double. The cost is negligible — `ComputedTrustScoreSource` calls `computeAll` on read (not "not a hot loop" as originally claimed), but the actual cost on the read path is the write-time snapshot lookup (D9 eliminates I/O entirely by pre-computing at write time).
**Sources:** AttestorCredibilityPolicy.CredibilityAssessment (pattern), ComputedTrustSourceCore (on-read path)
**Exploration:** quick
**Status:** revised (was: captured) — R1-08

## D4: Distance-to-weight mapping — distance-indexed weight table (eidos-owned)

**Choice:** Distance-to-weight mapping is eidos's concern (the implementation lives there — see D8). The default implementation uses a **distance-indexed weight table**: an ordered list of weights where index = org distance and the last entry repeats for all greater distances. Default table: `[1.0, 1.0, 0.7, 0.65, 0.6]` — meaning distance 0 (same agent) = 1.0, distance 1 (same unit) = 1.0, distance 2 (sibling units) = 0.7, distance 3 = 0.65, distance ≥ 4 = 0.6.

Special cases:
- `null` orgDistance (no write-time snapshot, no org structure) → NEUTRAL (1.0). This preserves current behaviour ("falls back to equal weighting when no org structure" — issue acceptance criterion).
- Distance 0 (same agent / self-attestation) → 1.0 (NEUTRAL). Self-attestation filtering is handled separately by `EigenTrustComputer` and should not be conflated with proximity weighting.
- Distance -1 (disconnected / unknown attestor) → NEUTRAL (1.0). Unknown attestors (system attestors, human reviewers, external auditors not in the org tree) must not be penalised. Only affirmatively known cross-unit distance reduces weight.
**Alternatives:**
- Logarithmic decay `max(floor, 1.0 - dropoff × ln(distance))` — over-parameterised for the actual use case. The defaults produced counterintuitive results: distance 2 gave 0.91 instead of the issue's specified 0.7, and the floor was only reached at distance ≥ 10. The issue asked for "cross-unit attestations weighted at 70%", which the table achieves directly at distance 2.
- Flat discount (everything cross-unit = 0.7) — expressible as a 3-entry table `[1.0, 1.0, 0.7]`. The table subsumes this.
- Exponential decay — over-penalises moderate distances in shallow org trees.
**Rationale:** A weight table is auditable (the reason names the exact entry and weight), matches the issue's intent directly, and can express any discrete mapping. It avoids the parameter-derivation complexity of continuous functions (dropoff, maxDistance, division-by-zero at maxDistance=1). Eidos configures it; ledger never sees it.
**Trade-offs:** Cannot express continuous curves. Not needed — org distance is discrete (integer hops). If a consumer needs a continuous function, they swap the entire eidos relevance factor.
**Sources:** Issue #231 acceptance criteria ("cross-unit attestations weighted at 70%"), eidos OrgAwareDispatch.orgDistance() (integer semantics), DefaultOrgAwareDispatch (LCA hop count), NoOpOrgAwareDispatch (always returns -1)
**Exploration:** deep-analysis
**Status:** revised (was: captured) — R1-02, R1-10, R1-11

## D5: Integration — per-attestation relevance via AttestationWeights parameter object

**Choice:** Introduce `AttestationWeights` parameter object holding `Map<String, AttestorCredibilityPolicy.CredibilityAssessment> credibilityByAttestorId` and `Map<UUID, Double> relevanceByAttestationId`. TrustScoreComputer.compute() and computeDimensionScore() gain new overloads taking `AttestationWeights` instead of growing the positional Map chain. Existing overloads delegate with `AttestationWeights.NEUTRAL` (empty maps). Relevance is keyed by attestation UUID (not attestorId) because the same attestor can have different relevance weights across tenants and over time (see D1, D9).

TrustScoreCalculator.computeAll() gains `scoredActorId` parameter. It batch-resolves both credibility and relevance, builds the `AttestationWeights` object, and passes it down. `PerActorTrustComputerCore` already has `actorId` — just passes it through.
**Alternatives:**
- Separate `Map<String, Double> relevanceByAttestorId` parameter (original D5) — wrong keying. Same attestor can attest in different tenants at different org distances. Per-attestor map loses this.
- Growing overload chain with another positional Map — accidentally passing credibility where relevance belongs is easy. The parameter object makes it type-safe.
- Merge into credibility map — conflates attestor-scoped and per-attestation-scoped concepts.
**Rationale:** The parameter object groups related weight lookups, prevents parameter swapping bugs, and provides `AttestationWeights.NEUTRAL` for tests. The per-attestation keying handles cross-tenant correctly: each attestation's relevance is resolved using the entry's tenancyId (available via `ledgerEntryId → decisions` mapping in TrustScoreCalculator).
**Trade-offs:** One new record class. Justified by type safety and correct cross-tenant semantics.
**Sources:** TrustScoreComputer.compute() overload chain, TrustScoreCalculator.computeAll() batch resolution, CrossTenantLedgerEntryRepository (cross-tenant data loading)
**Exploration:** deep-analysis
**Status:** revised (was: captured) — R1-03, R1-07

## D6: Config model — ledger has no relevance config; implementation config owned by eidos

**Choice:** Ledger ships the SPI (`AttestationRelevancePolicy`) and the composition pipeline (D7) with zero config. The default is NEUTRAL (no relevance factors active). Implementation-specific config (weight table, opt-in flags) is owned by the module that ships the implementation — for org proximity, that's eidos.

Eidos config (under `casehub.eidos.trust.relevance.*` or similar):
- Weight table: `casehub.eidos.trust.relevance.weights=1.0,1.0,0.7,0.65,0.6`
- The eidos module validates its own config at startup (weight table non-empty, all values in [0,1]).

Spring parity: eidos ships equivalent `@ConfigurationProperties` for Spring Boot alongside the Quarkus `@ConfigMapping`.
**Alternatives:**
- Ledger-owned config under `casehub.ledger.trust.relevance.*` — wrong module. Ledger doesn't own the implementation (D8). Config in ledger for an eidos-specific implementation couples the two.
- `enabled` flag in ledger — opt-in by classpath presence is the coherent default (matches credibility: no enable flag, NoOp @DefaultBean, real policy displaces by priority). An `enabled=false` flag only controls the default impl; a swapped-in SPI ignores it, making "disabled" a lie.
**Rationale:** Follows the dependency direction: config lives with the implementation it configures. Ledger doesn't know or care what implementations exist. The existing trust config namespace (`casehub.ledger.trust-score.*`, `casehub.ledger.decay.*`) stays clean.
**Trade-offs:** Consumers must look at the implementing module's docs for config keys, not ledger's. Acceptable — this is standard SPI practice.
**Sources:** LedgerConfig.java (existing trust config namespace), AttestorCredibilityPolicy (no enable flag, NoOp default), platform dependency-map.md (dependency direction)
**Exploration:** quick
**Status:** revised (was: captured) — R1-12

## D7: Extension model — ledger-owned factor pipeline via `Instance<AttestationRelevanceFactor>`

**Choice:** Ledger owns composition via CDI `Instance<AttestationRelevanceFactor>` pipeline. Each module contributes a named factor bean implementing `AttestationRelevanceFactor` (a simple sub-interface or identical to `AttestationRelevancePolicy`). Ledger iterates all discovered factors, computes the product of their weights with a global floor (configurable, default 0.5 — prevents compounding from driving weights to near-zero), and concatenates reasons.

With no factor beans present, the result is `RelevanceAssessment.NEUTRAL` — backward-compatible.

```java
public interface AttestationRelevanceFactor {
    RelevanceAssessment assess(AttestationRelevanceContext ctx);
    default Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) { ... }
    String name(); // "org-proximity", "capability-overlap", etc.
}
```

The `AttestationRelevancePolicy` SPI (D1) becomes the internal aggregation point: the default implementation iterates factors and combines. Consumers who want to replace the entire aggregation logic (not just add a factor) swap the policy itself.
**Alternatives:**
- Consumer-side composite (original D7) — doesn't work under CDI. A `CompositeRelevancePolicy` displaces the `@DefaultBean`, and cannot inject the default to wrap it. The consumer must re-resolve `OrgAwareDispatch` and re-read config manually. Also: two independently shipped factor modules (eidos org-proximity, another module's capability-overlap) cannot both be active without consumer glue code.
- No pipeline, just SPI swap — works for one factor, fails at two independent factors from different modules.
**Rationale:** Follows the ADR 0005 `LedgerEntryEnricher` pipeline precedent: when a second mechanism appears for the same lifecycle concern, an `Instance<>` pipeline is the right abstraction. The weight chain already has four multiplicative factors, and the declared future factors (capability overlap, collaboration recency, domain specialisation) make a fifth and sixth likely. Each module contributes its factor bean independently; no glue code needed.
**Trade-offs:** Slightly more framework than a bare SPI swap. Justified by the concrete multi-module composition requirement and the ADR 0005 precedent.
**Sources:** ADR 0005 (LedgerEntryEnricher pipeline), Instance<LedgerEntryEnricher> (existing pattern), TrustWeightedAgentStrategy (classpath-presence displacement)
**Exploration:** quick
**Status:** revised (was: captured) — R1-09

## D8: Dependency model — ledger ships SPI + NoOp; eidos ships the org-proximity implementation

**Choice:** Ledger ships `AttestationRelevancePolicy` SPI in `api` and a `NoOpAttestationRelevancePolicy` `@DefaultBean` in `runtime` that returns `NEUTRAL` for all assessments. Zero eidos dependency in ledger — not even nullable, not even as a type reference.

Eidos ships `OrgProximityRelevanceFactor implements AttestationRelevanceFactor` as `@ApplicationScoped @Priority(1)`. Eidos already depends on `casehub-ledger-api`, so no new dependency edges are created. The org-proximity factor uses `OrgAwareDispatch.orgDistance()` internally and owns the distance-to-weight config (D6).
**Alternatives:**
- Nullable `OrgAwareDispatch` in ledger-core constructor (original D8) — creates a compile-time `ledger → eidos-api` dependency, inverting the platform's published dependency direction (`eidos → ledger`, not the reverse). `OrgAwareDispatch` is a 4-method dispatch interface; depending on it brings dispatch concerns into trust scoring (ISP violation). It also risks a CDI dependency loop since eidos selection already depends on `TrustScoreSource`.
- Separate `casehub-ledger-eidos` module — unnecessary. The natural home for org-proximity code is eidos itself, which already has the dependency edge.
- Optional Maven dependency on eidos-api — still a compile-time coupling.
**Rationale:** Follows the established platform pattern exactly: ledger ships `AttestorCredibilityPolicy` SPI + `NoOpAttestorCredibilityPolicy` `@DefaultBean`; real policies come from consumer modules via classpath presence. Org knowledge stays in the module that owns the org model. Ledger stays domain-agnostic in fact, not just in name.
**Trade-offs:** The org-proximity implementation lives in a separate repo (eidos). This means #231 implementation spans two repos. Acceptable — this is the correct module boundary, and the implementation is small.
**Sources:** platform dependency-map.md (build order: ledger has no casehubio deps, eidos depends on ledger), AttestorCredibilityPolicy + NoOpAttestorCredibilityPolicy (SPI + NoOp pattern), TrustWeightedAgentStrategy (classpath-presence displacement)
**Exploration:** quick
**Status:** revised (was: captured) — R1-01

## D9: Write-time org distance snapshot — deterministic, auditable, I/O-free read path

**Choice:** Store the attestor-to-scored-actor org distance as a nullable `Integer` column (`org_distance`) on `LedgerAttestation`, populated at write time by an enricher in the eidos module. The relevance policy reads this snapshot field from the `AttestationRelevanceContext` (D1) and maps it to a weight using pure logic — no `OrgAwareDispatch` calls on the read path.

When no org structure exists (eidos not on classpath, or `NoOpOrgAwareDispatch` returning -1): the enricher is absent, `org_distance` is null, and the relevance factor returns NEUTRAL (D4).
**Alternatives:**
- Compute-time lookup (original implicit assumption) — calls `orgDistance()` against the current org tree at every trust computation. After an org reorg, all past attestations are retroactively re-evaluated against the new structure. An attestation by a same-unit peer at the time of the attestation becomes "cross-unit" after the peer moves, contradicting D4's premise that "shared operational context" is what matters. Also: the three `TrustScoreSource` strategies (Materialized, Cached, Computed) would diverge after a reorg with no attestation change, breaking `TrustScoreSourceContractTest`. And: `orgDistance()` does registry I/O, which lands on the read path for `ComputedTrustScoreSource`.
- Store attestor and actor `orgUnitId` instead of distance — more data but requires re-resolving the distance. The distance is what the weight function needs; store what you consume.
**Rationale:** Follows ADR 0005 ("Group Membership Snapshot at WorkItem Creation") which chose snapshot because exclusion "must be deterministic", repeated provider calls "multiply latency and failure surface", and the "audit trail must explain" the result. Also follows qhorus#505 which resolves `orgUnitId` from sender membership at write time. The snapshot makes trust computation a pure function of ledger data: deterministic, auditable (EU AI Art. 12), reproducible from the ledger alone, free of I/O on the read path, and consistent across all three TrustScoreSource strategies.
**Trade-offs:** A schema change (`org_distance` column on `ledger_attestation`). Existing attestations have null (no org context), which maps to NEUTRAL — no score disruption on upgrade. The enricher lives in eidos, so the write-time cost is one `orgDistance()` call per attestation persist.
**Sources:** ADR 0005 (snapshot precedent), qhorus#505 (org write-time resolution), TrustScoreSourceContractTest (source agreement contract), ComputedTrustSourceCore (on-read computation path)
**Exploration:** deep-analysis (review-driven)
**Status:** captured

## D10: Evidence semantics — mass-preserving normalisation

**Choice:** Normalise relevance weights per scored actor so that total evidence mass is unchanged: `Σ(raw × credibility × normalised_relevance) = Σ(raw × credibility)`. Relevance changes *whose* evidence dominates, never *how much* evidence exists.

Normalisation happens at the TrustScoreCalculator level (after batch-resolving raw relevance weights from factors, before passing to TrustScoreComputer). The normalisation factor is `Σ(raw_i × credibility_i) / Σ(raw_i × credibility_i × relevance_i)` applied uniformly to all relevance weights.

This applies consistently to all four passes: capability Beta, global Beta, dimension weighted-average, and capability×dimension.

`credibilityRetention` metric remains solely about credibility. A separate `relevanceEffect` metric (optional) can track the redistribution effect if needed for diagnostics.
**Alternatives:**
- Raw multiplicative discount (original D5 implicit assumption) — treats "less relevant" as "less evidence", shrinking scores toward the 0.5 prior. Actors reviewed mostly by cross-unit attestors converge more slowly and sit closer to 0.5 than identical performers reviewed in-unit. This creates a topology-dependent bias: a shared-service agent serving every unit, or an agent in a newly formed unit, is structurally penalised regardless of performance. `TrustGateService` thresholds and `TrustWeightedAgentStrategy` turn that gap into routing exclusion.
- Per-pass normalisation with different strategies — over-complex. Uniform normalisation gives consistent semantics: relevance is always a reweighting, never an evidence discount.
**Rationale:** The purpose of org proximity weighting is to shift whose judgment matters most, not to reduce total evidence. Same-unit peers' attestations should count more *relative to* cross-unit peers', but the total weight of evidence for the actor should not change. Without normalisation, in the Beta model, a uniform 0.7 discount on all attestations gives `(1+2.1)/(2+2.1) = 0.756` instead of `(1+3)/(2+3) = 0.80` for 3 SOUND — a 5.5% drop from topology alone. In the weighted-average (dimension) pass, a uniform discount cancels out entirely, so the same factor means "less evidence" in Beta and "no effect" in dimensions — inconsistent semantics.
**Trade-offs:** One normalisation pass adds O(N) over the attestation list. Negligible. Requires the Calculator to see all relevance weights before passing to Computer (already true — batch resolution precedes per-pass computation).
**Sources:** TrustScoreComputer.compute() (Beta model), TrustScoreComputer.computeDimensionScore() (weighted average), TrustGateService (threshold enforcement), TrustWeightedAgentStrategy (routing impact)
**Exploration:** deep-analysis (review-driven)
**Status:** captured
