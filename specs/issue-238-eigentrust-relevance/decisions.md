# Decisions — EigenTrust Relevance Weighting (#238)

## D1: Which factors to apply to EigenTrust edge weights

**Choice:** Three factors: decay x confidence x relevance. Not credibility.
**Alternatives:**
- Relevance only — simpler, but misses decay (stale edges dominate) and confidence (signal quality lost)
- Full four-factor chain (matching Beta path) — double-counts credibility because EigenTrust already handles attestor trust transitively via power iteration
**Rationale:** EigenTrust has no built-in model for time, signal quality, or domain relevance. These must be applied as edge weights. Credibility is redundant because power iteration naturally reduces the influence of low-trust attestors through transitive propagation. Applying it as an edge weight would penalise them twice.
**Trade-offs:** EigenTrust and Beta paths now differ in their factor chain (3 vs 4 factors). This is intentional — the algorithms have different properties and should use factors appropriate to their mechanics.
**Sources:** EigenTrustComputer.java (matrix building loop), TrustScoreComputer.java (Beta four-factor chain), consumer analysis (engine, agency, eidos, neocortex — all use opaque scalars via TrustScoreSource)
**Exploration:** deep-analysis
**Status:** captured

## D2: Trust scoring evolution roadmap

**Choice:** Three-stage evolution preserving the opaque scalar interface, plus a TrustScoreDetail side-channel SPI with TrustAlgorithm enum (BETA, EIGENTRUST, HYBRID).
**Alternatives:**
- Replace OptionalDouble with structured TrustProfile everywhere — breaking change to TrustScoreSource SPI, all consumers must update
- ScoreType enum expansion (add TRANSITIVE) — consumers must know which to request, doesn't solve calibration
**Rationale:** The opaque scalar is the right abstraction for most consumers (ranking, weighting). The calibration problem only surfaces for threshold gating, and can be solved transparently via fusion. The TrustScoreDetail companion method gives consumers opt-in access to derivation context without changing the primary API.
**Trade-offs:** Fusion function design (Stage 3) requires a large enough calibration set. Small networks fall back to identity mapping. TrustScoreDetail adds SPI surface area.
**Sources:** Engine TrustWeightedAgentStrategy (threshold gating), eidos SimpleAgentSelector (threshold + ranking), neocortex TrustWeightedCbrRecordStore (similarity weighting)
**Exploration:** deep-analysis
**Depends on:** D1 (Stage 1 applies the factors decided in D1)
**Status:** captured
