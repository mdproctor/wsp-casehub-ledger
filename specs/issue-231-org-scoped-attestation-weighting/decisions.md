# Decisions — #231 Org-Scoped Attestation Weighting

## D1: SPI abstraction level — pair-scoped relevance, not attestor-scoped credibility

**Choice:** New SPI `AttestationRelevancePolicy` with pair-scoped `assess(attestorId, scoredActorId, tenancyId)` signature. Lives in `api` module.
**Alternatives:**
- Fold into `AttestorCredibilityPolicy` — credibility is attestor-scoped (`assess(attestorId)`) with no context about who's being scored. Proximity is pair-scoped. Fundamental concept mismatch; would require breaking the SPI signature.
- Hop-count-only function `weight(int orgDistance)` — too narrow for future pair-scoped factors (capability overlap, collaboration recency). Each new factor would need its own SPI with no compositional structure.
**Rationale:** Credibility asks "is Y a reliable judge in general?" Relevance asks "is Y's judgment pertinent to X specifically?" These are independent concepts — same attestor can be highly credible but irrelevant to a particular actor. The pair-scoped contract accommodates org proximity today and capability overlap, collaboration recency, domain specialisation tomorrow without changing the interface.
**Trade-offs:** Slightly more complex integration than a bare double function — requires batch resolution keyed by (attestor, actor) pairs. Justified by extensibility and diagnostic value of RelevanceAssessment.reason.
**Sources:** TrustScoreComputer.java (weight chain), AttestorCredibilityPolicy.java (SPI pattern), eidos OrgAwareDispatch.java (pair-scoped distance API)
**Exploration:** deep-analysis
**Status:** captured

## D2: Naming — `AttestationRelevancePolicy`, not `ProximityWeightFunction`

**Choice:** Name the SPI for the concept (relevance), not the first implementation (proximity).
**Alternatives:**
- `ProximityWeightFunction` — names the first consumer, not the abstraction. Would become misleading when capability overlap or collaboration recency implementations arrive.
- `AttestationContextPolicy` — too vague; "context" doesn't convey the weight-modifying role.
**Rationale:** Follows the existing pattern: `DecayFunction` (concept) not `ExponentialTimeWeight` (implementation). `AttestorCredibilityPolicy` (concept) not `TrustBasedCredibility` (implementation).
**Trade-offs:** Slightly less obvious what the SPI does at first glance vs a name that says "proximity." Mitigated by Javadoc.
**Sources:** DecayFunction.java, AttestorCredibilityPolicy.java (naming precedent)
**Exploration:** quick
**Status:** captured

## D3: Return type — `RelevanceAssessment(weight, reason)` record

**Choice:** Return a `RelevanceAssessment` record with `weight` (double) and `reason` (nullable String), plus static `NEUTRAL` constant.
**Alternatives:**
- Bare `double` — simpler but no diagnostics. Can't answer "why was this attestation downweighted?" in audit trails or compliance reports.
- Full record with flags (like CredibilityAssessment) — no concrete flag use case yet. Add flags when needed; the record is extensible.
**Rationale:** Follows `CredibilityAssessment(weight, reason, flags)` pattern minus flags (YAGNI). The `reason` string enables diagnostics ("org-distance:3, weight:0.82") and audit context without over-engineering.
**Trade-offs:** Marginally more allocation than a bare double. Negligible in context — trust computation is not a hot loop (runs nightly or on attestation events).
**Sources:** AttestorCredibilityPolicy.CredibilityAssessment (pattern)
**Exploration:** quick
**Status:** captured

## D4: Distance-to-weight mapping — logarithmic decay with configurable floor

**Choice:** `weight = max(floor, 1.0 - dropoff × ln(distance))` for distance > 1. Distance ≤ 1 → 1.0. Distance -1 (disconnected) → floor. Dropoff derived: `(1.0 - floor) / ln(maxDistance)`.
**Alternatives:**
- Flat discount (same-unit = 1.0, everything else = 0.7) — loses information about intermediate distances. Treatable as degenerate case of log curve with maxDistance=2.
- Exponential decay `base^(distance-1)` — faster falloff than log. Over-penalises moderate distances in shallow org trees.
- Linear decay — doesn't match how shared context actually drops off in hierarchies (biggest loss is first hop, diminishing returns after).
**Rationale:** First principles: shared operational context drops off logarithmically in a hierarchy. The jump from same-unit to sibling-unit (distance 1→2) is the biggest loss — different tools, different training data, different daily context. Subsequent hops (2→3, 3→4) lose progressively less marginal context. Logarithmic decay captures this naturally. The floor ensures cross-unit attestations always retain meaningful weight (accuracy-biased default, independence preserved).
**Trade-offs:** Slightly more complex than flat discount. Mitigated by: (a) flat discount is trivially expressible via config (maxDistance=2), (b) the formula is documented in Javadoc, (c) pure function — easy to unit test with property-based tests.
**Sources:** First-principles analysis of information flow in hierarchical organisations and agent topologies
**Exploration:** deep-analysis
**Status:** captured

## D5: Integration — fourth multiplicative factor via method overloads

**Choice:** Add `Map<String, Double> relevanceByAttestorId` parameter to TrustScoreComputer.compute() and computeDimensionScore() as new overloads. Existing overloads delegate with Map.of() — fully backward-compatible. TrustScoreCalculator.computeAll() gains `scoredActorId` parameter and `AttestationRelevancePolicy` constructor dependency.
**Alternatives:**
- Generic weight modifier pipeline — over-engineering for two factors (credibility, relevance). Java method overloading already provides additive extension.
- Merge into credibility map — conflates attestor-scoped and pair-scoped concepts. Loses the ability to reason about them independently.
**Rationale:** Follows the existing pattern exactly. Credibility was added the same way — new map parameter, new overloads, batch resolution in Calculator. The effective weight chain becomes `decay × confidence × credibility × relevance`. Each factor is independently resolvable, testable, and configurable.
**Trade-offs:** Method signature grows by one parameter. Acceptable — the overload chain ensures old callers don't break, and the pattern is established.
**Sources:** TrustScoreComputer.compute() overload chain, TrustScoreCalculator.computeAll() batch resolution pattern
**Exploration:** deep-analysis
**Status:** captured

## D6: Config model — three keys under `casehub.ledger.trust.relevance.*`

**Choice:** `enabled` (boolean, default true), `cross-unit-discount` (double, default 0.7), `max-distance` (int, default 10). Config tunes parameters; SPI swap changes algorithm.
**Alternatives:**
- Single flat discount config — doesn't support distance-sensitive decay.
- Enum-based mode selector (flat/log/exponential) — config shouldn't select algorithms. That's the SPI's job.
**Rationale:** Three knobs cover the full tuning surface of the default implementation. enabled=false makes the policy return NEUTRAL everywhere (disable without removing the SPI). cross-unit-discount sets the floor. max-distance controls the decay curve shape. Flat discount is achievable via max-distance=2.
**Trade-offs:** None significant. Config keys are cheap and self-documenting via @ConfigMapping Javadoc.
**Sources:** LedgerConfig.java (config pattern), ExponentialDecayFunction (config-driven SPI default)
**Exploration:** quick
**Status:** captured

## D7: Extension model — consumer-side composite policy

**Choice:** Future pair-scoped factors compose via `CompositeRelevancePolicy implements AttestationRelevancePolicy` on the consumer side. If a factor is conceptually distinct enough to warrant separation, it gets a new SPI + new map parameter following the same additive overload pattern.
**Alternatives:**
- Framework-level pipeline abstraction — premature. Two factors (credibility, relevance) don't justify a pipeline. If we reach three pair-scoped factors, revisit.
- Plugin/extension-point registry — over-engineering. CDI alternatives already provide SPI swapping.
**Rationale:** Interface composition is the simplest mechanism that works. A consumer needing capability-overlap + org-proximity writes one class that wraps both and multiplies weights. No framework needed.
**Trade-offs:** Consumer must write the composite class manually. Acceptable — this is CDI-standard practice and the composite is ~20 lines.
**Sources:** AttestorCredibilityPolicy (SPI swap via CDI alternatives), DecayFunction (same pattern)
**Exploration:** quick
**Status:** captured

## D8: Dependency model — ledger must not hard-depend on eidos

**Choice:** `AttestationRelevancePolicy` SPI in `api` has zero eidos dependency. `OrgProximityRelevancePolicy` default implementation in `ledger-core` takes nullable `OrgAwareDispatch` — null means no org structure, returns NEUTRAL everywhere. CDI producer in `runtime` uses `Instance<OrgAwareDispatch>` to resolve optionally.
**Alternatives:**
- Optional Maven dependency on eidos-api — still couples the core module. Nullable constructor parameter is simpler.
- Separate module `casehub-ledger-eidos` — over-engineering for a single nullable dependency.
**Rationale:** Domain-agnostic ledger principle. The SPI is about relevance, not org structure. The default impl happens to use org distance but degrades gracefully. Consumers without eidos on the classpath get current behaviour with zero config.
**Trade-offs:** OrgAwareDispatch type appears in ledger-core as a constructor parameter. Acceptable — it's nullable and the class is internal (not SPI surface).
**Sources:** casehub-ledger design principles (domain-agnostic), NoOpOrgAwareDispatch pattern in eidos
**Exploration:** quick
**Status:** captured
