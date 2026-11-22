## D1: Unit grouping granularity

**Choice:** org-distance threshold (distance ≤ N, default 1)
**Alternatives:**
- Explicit unit-id in snapshot — precise grouping but requires eidos enricher changes and a new snapshot key
- Configurable distance bands — multiple independence tiers with per-band caps; more nuanced but complex to configure and reason about
**Rationale:** The existing `org-distance` snapshot key already captures proximity. Distance ≤ 1 covers same-unit and parent/child pairs — imprecise but sufficient for a cap. No eidos changes needed.
**Trade-offs:** Parent/child pairs are treated as "same unit" even though they may be semi-independent. Acceptable for a cap that limits over-representation rather than eliminating it.
**Sources:** `eidos/org-runtime/.../OrgProximityRelevanceFactor.java` (reads `org-distance`), `eidos/org-runtime/.../OrgDistanceAttestationEnricher.java` (writes `org-distance`), issue-231 design spec §Default Implementation
**Exploration:** quick
**Status:** captured

## D2: Module home

**Choice:** ledger-core
**Alternatives:**
- eidos org-runtime — co-located with OrgProximityRelevanceFactor, but couples a generic concept to org-specific implementation
**Rationale:** Anti-collusion capping is a generic trust computation concern. The factor reads `org-distance` from the snapshot like any factor reading enricher data — no eidos dependency. Activates by configuration.
**Trade-offs:** Factor has no access to eidos org APIs — limited to snapshot data. Acceptable because snapshot captures the relationship that existed at attestation time (write-time snapshot principle from ADR 0005).
**Sources:** `api/spi/AttestationRelevanceFactor.java` (SPI contract), `ledger-core/.../RelevanceComposer.java` (composition), issue-231 design spec §Architecture
**Exploration:** quick
**Status:** captured

## D3: Group-level reasoning mechanism

**Choice:** Override assessBatch() on the SPI; change RelevanceComposer to call assessBatch() instead of assess()
**Alternatives:**
- Post-factor cap in RelevanceComposer — simpler but couples the independence concept to the composer, reducing extensibility
- Pre-computed group stats in context — requires a new pre-assessment pass in TrustScoreCalculator and adds group-level data to a per-attestation record
**Rationale:** assessBatch() already exists on the SPI as a default method that delegates to assess(). Switching RelevanceComposer to call it is a minimal API change. Existing factors work unchanged via the default delegation. The independence factor overrides assessBatch() to reason about group composition — the natural extension point.
**Trade-offs:** Factors that override assessBatch() must handle the full batch correctly. The default delegation ensures backward compatibility.
**Sources:** `api/spi/AttestationRelevanceFactor.java:36-42` (assessBatch default method), `ledger-core/.../RelevanceComposer.java:42-52` (current per-attestation loop)
**Exploration:** quick
**Status:** captured

## D4: Cap mechanics

**Choice:** Proportional down-weight when same-unit share exceeds cap
**Alternatives:**
- Hard cap with redistribution — excess redistributed to cross-unit attestors; creates artificial boosts to attestors who may not deserve them
- Diversity bonus for cross-unit — additive rather than subtractive; mathematically similar after normalisation but doesn't directly address the over-representation problem
**Rationale:** Smooth, proportional, no cliff effects. When proximate share exceeds the cap (e.g. 0.6), scale proximate weights down by `cap / share`. Cross-unit weights stay at 1.0. The cap is a ceiling on influence share, not a hard cutoff.
**Trade-offs:** When ALL attestations are proximate, the cap scales all equally — after mass-preserving normalisation, the effect cancels. The cap only bites in mixed groups, which is the intended behaviour.
**Sources:** ARC42STORIES.MD line 1313 (in-unit amplification risk), issue-231 design spec §Motivation (accuracy vs independence tension)
**Exploration:** quick
**Status:** captured

## D5: Factor interaction model

**Choice:** Independent composition — each factor reasons about one concern, RelevanceComposer multiplies
**Alternatives:**
- Aware of org-proximity weights — reads org-proximity's weights to cap EFFECTIVE share; more precise but creates ordering dependency between factors
**Rationale:** Clean separation. The independence factor operates on raw group counts (how many attestations from org-distance ≤ N), not on weights that org-proximity already assigned. No ordering dependency. Each factor is independently testable.
**Trade-offs:** The combined effect of org-proximity boost × independence cap may not exactly match the configured cap threshold. Acceptable because mass-preserving normalisation in TrustScoreComputer adjusts the final effective weights anyway.
**Sources:** `ledger-core/.../RelevanceComposer.java` (product-with-floor composition), `ledger-core/.../TrustScoreComputer.java:110-138` (mass-preserving normalisation)
**Exploration:** quick
**Status:** captured
