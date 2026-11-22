## D1: Unit grouping granularity

**Choice:** org-distance == 1 (hardcoded, same OrgUnit)
**Alternatives:**
- Configurable threshold (distance ≤ N, default 1) — more flexible but blurs the concept and adds config surface; rejected in companion eidos issue-257 D2
- Explicit unit-id in snapshot — precise grouping but requires eidos enricher changes and a new snapshot key
- Configurable distance bands — multiple independence tiers with per-band caps; more nuanced but complex to configure and reason about
**Rationale:** Distance == 1 is a crisp, well-defined concept: members of the same OrganizationalUnit. This is where operational context overlap is maximal and evidence correlation is highest. Distance 2+ attestors are in structurally separate units with different observation vantage points. The companion eidos issue-257 D2 investigation concluded that a configurable threshold "blurs the concept and adds config surface." Distance 0 (self-attestation) is excluded by `TrustScoreComputer` before factors run, so the == vs ≤ distinction at N=1 is immaterial.
**Trade-offs:** Parent/child pairs (distance 2) are treated as cross-unit even though they may share some context. Sibling-unit correlation (distance 2) is not captured. Acceptable because the concern is specifically intra-unit amplification.
**Sources:** `eidos/org-runtime/.../IndependenceAwareRelevanceFactor.java` (distance == 1 check), `eidos/org-runtime/.../OrgDistanceAttestationEnricher.java` (writes `org-distance`), eidos issue-257 D2 (companion decision rejecting configurable threshold)
**Exploration:** quick
**Status:** revised — changed from "distance ≤ N, default 1" to "distance == 1 (hardcoded)" to match companion eidos decisions and implementation

## D2: Module home

**Choice:** eidos org-runtime
**Alternatives:**
- ledger-core — anti-collusion capping is conceptually generic, but the current factor is semantically coupled to org-distance (distance == 1 defines "same-unit"), which is an eidos concept. A truly generic independence factor in ledger-core would need an abstract grouping mechanism (group-identity keys), not distance thresholds — a different and larger design.
**Rationale:** The factor references `OrgProximityRelevanceFactor.SNAPSHOT_KEY` and interprets distance == 1 as "same OrgUnit" — both are eidos-specific semantics. ADR 0018 states "concrete AttestationRelevanceFactor implementations live in consumers, not in ledger." The issue-231 design spec §Architecture explicitly places the factor in eidos. The compile-time reference to `SNAPSHOT_KEY` is secondary — the real dependency is semantic coupling to the org-distance concept.
**Trade-offs:** Factor has a compile-time coupling to `OrgProximityRelevanceFactor` via `SNAPSHOT_KEY`. Could be broken by inlining the string literal, but the semantic coupling (distance == 1 means "same unit") is the real dependency — not the constant reference.
**Sources:** `eidos/org-runtime/.../IndependenceAwareRelevanceFactor.java` (actual location), ADR 0018 §Consequences ("implementations live in consumers"), issue-231 design spec §Architecture (shows factor in eidos)
**Exploration:** quick
**Status:** revised — changed from "ledger-core" to "eidos org-runtime" to match ADR 0018, issue-231 design spec, and actual implementation

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
- Effective sample size (ESS) — when N observations have pairwise correlation ρ, ESS = N/(1+(N-1)ρ); theoretically principled but requires correlation estimation between attestors, which the platform cannot perform without ground truth data
- Unit-level aggregation — aggregate same-unit attestations into a single unit-level opinion, then combine unit opinions as independent observations; principled but discards within-unit variation (8 agreeing attestors vs 1 is meaningful information the aggregate loses)
**Rationale:** When same-unit share exceeds the cap, compute per-attestation weight to hit the target share exactly: `w = cap × crossCount / ((1 − cap) × sameCount)`. Cross-unit weights stay at 1.0. The cap is a ceiling on influence share, not a hard cutoff. Smooth, proportional, no cliff effects. The cap produces exact results when the independence factor is the sole factor affecting weights; when composed with other factors, the effective share may deviate from the configured cap (see D5).
**Trade-offs:** When ALL attestations are proximate, the cap is not triggered (no cross-unit attestors to protect) — after mass-preserving normalisation, all weights return to baseline. The cap only bites in mixed groups, which is the intended behaviour.
**Sources:** ARC42STORIES.MD line 1313 (in-unit amplification risk), issue-231 design spec §Motivation (accuracy vs independence tension), eidos issue-257 D3 (derives the formula algebraically)
**Exploration:** quick
**Status:** revised — corrected formula description from imprecise "cap / share" to the actual implemented formula `cap × crossCount / ((1 − cap) × sameCount)`; added ESS and unit-level aggregation to rejected alternatives

## D5: Factor interaction model

**Choice:** Independent composition — each factor reasons about one concern, RelevanceComposer multiplies
**Alternatives:**
- Aware of org-proximity weights — reads org-proximity's weights to cap EFFECTIVE share; more precise but creates ordering dependency between factors and couples the independence factor to the existence and weighting strategy of the org-proximity factor
**Rationale:** Clean separation. The independence factor operates on raw group counts (how many attestations from org-distance == 1), not on weights that org-proximity already assigned. No ordering dependency. Each factor is independently testable. The cap produces exact results in isolation. When composed with other factors, the effective same-unit share deviates from the configured cap — the deviation is bounded by the other factors' weight range (org-proximity's lowest weight is 0.6 at distance ≥ 4).
**Trade-offs:** The cap is approximate when other factors produce non-uniform weights. For the current org-proximity factor (weights 1.0, 0.7, 0.65, 0.6), the worst-case deviation occurs when cross-unit attestors are at maximum distance (weight 0.6), producing an effective same-unit share above the configured cap. Mass-preserving normalisation in `TrustScoreComputer` preserves total evidence mass but does not adjust share ratios between groups — it scales all attestations by the same normFactor.
**Sources:** `ledger-core/.../RelevanceComposer.java` (product-with-floor composition), `ledger-core/.../TrustScoreComputer.java:110-138` (mass-preserving normalisation)
**Exploration:** quick
**Status:** revised — removed incorrect claim that mass-preserving normalisation adjusts effective weights to match the cap; it preserves total mass but share ratios are unchanged by normalisation

## D6: Share computation basis

**Choice:** Attestation count — share = sameCount / (sameCount + crossCount)
**Alternatives:**
- Evidence mass — use base weights (decay × confidence × credibility) to compute share; more accurate representation of actual evidence influence but requires the factor to access base weights, which are computed in `TrustScoreComputer` after `RelevanceComposer` has already produced relevance weights — not available through the `AttestationRelevanceContext` SPI
- Hybrid — count-triggered, mass-adjusted; complex with unclear benefit
**Rationale:** The factor SPI provides `AttestationRelevanceContext` records without base weights (decay, confidence, credibility). These weights are computed in `TrustScoreComputer` during the trust computation pass, after `RelevanceComposer` has already produced relevance weights. A mass-based cap would require extending `AttestationRelevanceContext` to carry pre-computed base weights and restructuring the computation pipeline in `TrustScoreCalculator` and `TrustScoreComputer`. Count-based share is a pragmatic approximation that works within the existing SPI boundary.
**Trade-offs:** Count-based share can over-trigger (penalising same-unit attestations already decayed to negligible weight) and under-trigger (failing to cap when same-unit attestations have disproportionately high base mass due to recency). These edge cases are bounded by the decay function's half-life.
**Sources:** `eidos/org-runtime/.../IndependenceAwareRelevanceFactor.java:68-69` (count-based fraction), `api/.../AttestationRelevanceContext.java` (no base weight field), `ledger-core/.../TrustScoreComputer.java` (base weight computation in trust pass)
**Exploration:** surfaced by review (R1-07)
**Status:** captured

## D7: Unknown attestation handling

**Choice:** Exclude unknown attestations from share denominator — share = sameCount / (sameCount + crossCount), unknowns not counted
**Alternatives:**
- Include unknowns in denominator — share = sameCount / totalCount; more lenient cap (harder to trigger), but counts unclassifiable attestations as evidence of diversity
- Treat unknowns as same-unit — most conservative, but penalises the scored actor for the enricher's inability to classify
- Treat unknowns as cross-unit — most lenient, but could allow same-unit over-representation to go uncapped
**Rationale:** Unknown attestations lack org-distance data — their org relationship cannot be determined. Counting them in either bucket is arbitrary. Excluding them gives the cap a clear semantic: "of the attestations whose org relationship is known, what fraction are same-unit?" This makes the cap more aggressive (easier to trigger) because the denominator is smaller, but it's honest about what it measures. When crossCount == 0 or sameCount == 0, all attestations receive NEUTRAL — preventing the cap from triggering on degenerate distributions.
**Trade-offs:** When many attestations are unknown (e.g. org-distance enricher not deployed), the cap operates on a small denominator and may trigger on small samples. Mitigated by the guard clause: when crossCount == 0 or sameCount == 0, all attestations receive NEUTRAL.
**Sources:** `eidos/org-runtime/.../IndependenceAwareRelevanceFactor.java:55-66` (unknown handling, guard clause), eidos issue-257 D2 (companion decision on same-unit definition excluding unknowns)
**Exploration:** surfaced by review (R1-08)
**Status:** captured
