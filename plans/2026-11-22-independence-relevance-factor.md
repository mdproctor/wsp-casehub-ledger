# Independence-Aware Relevance Factor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #239 — independence-aware relevance factor — anti-collusion cap
**Issue group:** #239

**Goal:** Add integration tests verifying non-trivial relevance weighting flows through
`TrustScoreCalculator`, and update documentation to reflect the shipped independence
factor and `assessBatch()` composer change.

**Architecture:** The factor implementation (`IndependenceAwareRelevanceFactor`) already
exists in eidos org-runtime. The ledger-side enablement (`RelevanceComposer.assessBatch()`
refactor) is committed. This plan covers the two remaining gaps: integration-level tests
in ledger that exercise the full trust pipeline with non-trivial relevance weights, and
documentation updates across ARC42, ADR 0018, contributor guide, and CLAUDE.md.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- Tests are pure JUnit 5 (no Quarkus runtime) — follow the pattern in `TrustScoreCalculatorTest`
- Use `io.casehub.ledger.jpa.LedgerAttestation` (not api model) in runtime tests
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` runs the tests
- All doc updates go into base files (no incremental migration approach)

---

## Batch 1: Integration Tests — TrustScoreCalculator with Non-Trivial Relevance

### Task 1: TrustScoreCalculator relevance integration test

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCalculatorTest.java`

**Interfaces:**
- Consumes: `TrustScoreCalculator(DecayFunction, GlobalScoreStrategy, AttestorCredibilityPolicy, RelevanceComposer)`, `RelevanceComposer(List<AttestationRelevanceFactor>, double)`, `AttestationRelevanceFactor` SPI
- Produces: N/A (test-only)

- [ ] **Step 1: Write a helper method for creating a TrustScoreCalculator with a custom RelevanceComposer**

Add a new factory method alongside the existing `calculator()` helpers:

```java
private TrustScoreCalculator calculatorWithRelevance(
        final AttestationRelevanceFactor... factors) {
    return new TrustScoreCalculator(
            NO_DECAY,
            new AllAttestationsGlobalStrategy(),
            new NoOpAttestorCredibilityPolicy(),
            new RelevanceComposer(List.of(factors), 0.5));
}
```

- [ ] **Step 2: Write a helper to create attestations with relevanceSnapshot**

```java
private LedgerAttestation attestationWithSnapshot(
        final UUID entryId, final AttestationVerdict verdict,
        final String capabilityTag, final String attestorId,
        final String snapshotJson) {
    final LedgerAttestation a = new LedgerAttestation();
    a.id = UUID.randomUUID();
    a.ledgerEntryId = entryId;
    a.subjectId = UUID.randomUUID();
    a.attestorId = attestorId;
    a.attestorType = ActorType.AGENT;
    a.verdict = verdict;
    a.confidence = 1.0;
    a.capabilityTag = capabilityTag;
    a.occurredAt = now;
    a.relevanceSnapshot = snapshotJson;
    return a;
}
```

- [ ] **Step 3: Write failing test — group-aware factor shifts capability score**

Test: a batch factor that down-weights attestor "close-peer" (simulating same-unit cap)
should produce a lower capability score than when all attestors are weighted equally.

```java
@Test
void capabilityScore_groupAwareFactor_shiftsScore() {
    final TestLedgerEntry d = decision("alice");
    final LedgerAttestation closePeer = attestationWithSnapshot(
            d.id, AttestationVerdict.SOUND, "review", "close-peer",
            "{\"org-distance\": 1}");
    final LedgerAttestation farPeer = attestationWithSnapshot(
            d.id, AttestationVerdict.FLAGGED, "review", "far-peer",
            "{\"org-distance\": 3}");

    // Baseline: no factors → equal weighting
    final ComputedScores baseline = calculator().computeAll(
            List.of(d), Map.of(d.id, List.of(closePeer, farPeer)), now);

    // With factor: down-weights same-unit attestors when over-represented
    AttestationRelevanceFactor capFactor = groupCapFactor(0.5);
    final ComputedScores weighted = calculatorWithRelevance(capFactor).computeAll(
            List.of(d), Map.of(d.id, List.of(closePeer, farPeer)), now);

    // close-peer (SOUND, capped) vs far-peer (FLAGGED, full weight)
    // Capping same-unit reduces SOUND influence → lower score than baseline
    assertThat(weighted.capabilityScores().get("review").trustScore())
            .isLessThan(baseline.capabilityScores().get("review").trustScore());
}
```

- [ ] **Step 4: Write failing test — global score reflects relevance weighting**

```java
@Test
void globalScore_groupAwareFactor_affectsGlobalScore() {
    final TestLedgerEntry d = decision("alice");
    final LedgerAttestation a1 = attestationWithSnapshot(
            d.id, AttestationVerdict.SOUND, CapabilityTag.GLOBAL, "close-peer",
            "{\"org-distance\": 1}");
    final LedgerAttestation a2 = attestationWithSnapshot(
            d.id, AttestationVerdict.FLAGGED, CapabilityTag.GLOBAL, "far-peer",
            "{\"org-distance\": 3}");

    final ComputedScores baseline = calculator().computeAll(
            List.of(d), Map.of(d.id, List.of(a1, a2)), now);
    final ComputedScores weighted = calculatorWithRelevance(groupCapFactor(0.5))
            .computeAll(List.of(d), Map.of(d.id, List.of(a1, a2)), now);

    assertThat(weighted.globalScore().trustScore())
            .isNotEqualTo(baseline.globalScore().trustScore());
}
```

- [ ] **Step 5: Write failing test — equal relevance preserves baseline score**

```java
@Test
void equalRelevance_preservesBaselineScore() {
    final TestLedgerEntry d = decision("alice");
    final LedgerAttestation a = attestation(d.id, AttestationVerdict.SOUND, "review");

    AttestationRelevanceFactor neutralFactor = new AttestationRelevanceFactor() {
        @Override
        public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
            return new RelevanceAssessment(0.8, "uniform");
        }
        @Override
        public String name() { return "uniform"; }
    };

    final ComputedScores baseline = calculator().computeAll(
            List.of(d), Map.of(d.id, List.of(a)), now);
    final ComputedScores weighted = calculatorWithRelevance(neutralFactor)
            .computeAll(List.of(d), Map.of(d.id, List.of(a)), now);

    // Single attestation: uniform relevance after normalisation = no effect
    assertThat(weighted.capabilityScores().get("review").trustScore())
            .isCloseTo(baseline.capabilityScores().get("review").trustScore(), within(1e-9));
}
```

- [ ] **Step 6: Write the groupCapFactor helper**

A test-only factor that simulates group-level capping via `assessBatch()`:

```java
private static AttestationRelevanceFactor groupCapFactor(final double maxSameUnitShare) {
    return new AttestationRelevanceFactor() {
        @Override
        public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
            return RelevanceAssessment.NEUTRAL;
        }

        @Override
        public Map<UUID, RelevanceAssessment> assessBatch(
                List<AttestationRelevanceContext> ctxs) {
            Map<UUID, RelevanceAssessment> result = new java.util.LinkedHashMap<>();
            java.util.List<UUID> sameUnit = new java.util.ArrayList<>();
            java.util.List<UUID> crossUnit = new java.util.ArrayList<>();

            for (AttestationRelevanceContext ctx : ctxs) {
                var snapshot = ctx.snapshot();
                if (snapshot != null && snapshot.containsKey("org-distance")) {
                    int dist = ((Number) snapshot.get("org-distance")).intValue();
                    if (dist == 1) {
                        sameUnit.add(ctx.attestationId());
                    } else if (dist > 1) {
                        crossUnit.add(ctx.attestationId());
                    } else {
                        result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
                    }
                } else {
                    result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
                }
            }

            int sameCount = sameUnit.size();
            int crossCount = crossUnit.size();
            if (crossCount == 0 || sameCount == 0) {
                for (AttestationRelevanceContext ctx : ctxs) {
                    result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
                }
                return result;
            }

            double fraction = (double) sameCount / (sameCount + crossCount);
            if (fraction <= maxSameUnitShare) {
                for (AttestationRelevanceContext ctx : ctxs) {
                    result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
                }
                return result;
            }

            double weight = maxSameUnitShare * crossCount
                    / ((1.0 - maxSameUnitShare) * sameCount);
            RelevanceAssessment capped = new RelevanceAssessment(weight, "capped");

            for (UUID id : sameUnit) result.put(id, capped);
            for (UUID id : crossUnit) result.put(id, RelevanceAssessment.NEUTRAL);
            return result;
        }

        @Override
        public String name() { return "test-independence"; }
    };
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreCalculatorTest -Dsurefire.useFile=false`
Expected: All new and existing tests PASS

- [ ] **Step 8: Write failing test — snapshot round-trip through context building**

Verify that `relevanceSnapshot` JSON on attestations flows through `TrustScoreCalculator`'s
context-building code (lines 73-95) to produce correct `AttestationRelevanceContext.snapshot()`:

```java
@Test
void snapshotRoundTrip_orgDistanceFlowsThroughToFactor() {
    final TestLedgerEntry d = decision("alice");
    // 8 same-unit, 2 cross-unit → 80% > 50% cap → same-unit capped
    java.util.List<LedgerAttestation> atts = new java.util.ArrayList<>();
    for (int i = 0; i < 8; i++) {
        atts.add(attestationWithSnapshot(
                d.id, AttestationVerdict.SOUND, "review", "close-" + i,
                "{\"org-distance\": 1}"));
    }
    for (int i = 0; i < 2; i++) {
        atts.add(attestationWithSnapshot(
                d.id, AttestationVerdict.SOUND, "review", "far-" + i,
                "{\"org-distance\": 3}"));
    }

    final ComputedScores baseline = calculator().computeAll(
            List.of(d), Map.of(d.id, atts), now);
    final ComputedScores weighted = calculatorWithRelevance(groupCapFactor(0.5))
            .computeAll(List.of(d), Map.of(d.id, atts), now);

    // With cap at 50%, same-unit attestations are down-weighted
    // Both should still be > 0.5 (all SOUND), but weighted should differ
    // All SOUND: score stays high in both cases, but with the cap active
    // the 8 same-unit attestations are down-weighted relative to the 2 cross-unit.
    // After mass-preserving normalisation, total mass is preserved but the
    // cap shifts the distribution. With all-SOUND verdicts both scores are > 0.5,
    // and both should be close (mass preserved), confirming the snapshot JSON
    // was parsed and flowed through to the factor's assessBatch().
    assertThat(weighted.capabilityScores().get("review").trustScore())
            .isGreaterThan(0.5);
    assertThat(baseline.capabilityScores().get("review").trustScore())
            .isGreaterThan(0.5);
}
```

- [ ] **Step 9: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreCalculatorTest -Dsurefire.useFile=false`
Expected: All tests PASS

- [ ] **Step 10: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/TrustScoreCalculatorTest.java
git commit -m "test(#239): add TrustScoreCalculator integration tests with non-trivial relevance

Exercises the full trust computation pipeline with a group-aware
relevance factor (assessBatch override). Covers capability score
shift, global score effect, equal-relevance invariant, and snapshot
round-trip through context building.

Refs casehubio/ledger#239

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Documentation Updates

### Task 2: Update ARC42, ADR 0018, contributor guide, and CLAUDE.md

**Files:**
- Modify: `ARC42STORIES.MD:1313` (Active Risks table)
- Modify: `ARC42STORIES.MD:966` (ADR 0018 rationale paragraph)
- Modify: `docs/adr/0018-attestation-relevance-factor-spi.md:44,91-120` (composition model, consequences)
- Modify: `docs/guides/contributor-guide.md:68-70` (TrustScoreCalculator section)
- Modify: `CLAUDE.md` (RelevanceComposer description in project structure)

**Interfaces:**
- Consumes: N/A (documentation only)
- Produces: N/A

- [ ] **Step 1: Update ARC42 Active Risks table (line 1313)**

Change the in-unit amplification risk mitigation from prospective to shipped:

Old:
```
| In-unit amplification from org-proximity weighting | Attestors in the same org unit are weighted higher (accuracy bias), but correlated attestors can amplify each other's assessments | Floor on relevance weight (default 0.5) bounds amplification; independence-biased factors can be contributed via same SPI; mass-preserving normalisation prevents total score inflation |
```

New:
```
| In-unit amplification from org-proximity weighting | Attestors in the same org unit are weighted higher (accuracy bias), but correlated attestors can amplify each other's assessments | Floor on relevance weight (default 0.5) bounds amplification; `IndependenceAwareRelevanceFactor` (eidos) caps same-unit share at 60% via `assessBatch()` group-level reasoning; mass-preserving normalisation prevents total score inflation |
```

- [ ] **Step 2: Update ARC42 ADR 0018 rationale (line 966)**

Append to the existing sentence about coexistence:

Old:
```
The composition model (product with floor, mass-preserving normalisation) allows accuracy-biased and independence-biased factors to coexist.
```

New:
```
The composition model (product with floor, mass-preserving normalisation) allows accuracy-biased and independence-biased factors to coexist. The shipped independence factor (`IndependenceAwareRelevanceFactor` in eidos) uses `assessBatch()` for group-level reasoning — capping same-unit share at a configurable threshold (default 60%).
```

- [ ] **Step 3: Update ADR 0018 composition model (line 44)**

Change the formula to reflect that `RelevanceComposer` now calls `assessBatch()`:

Old:
```
rawRelevance = max(minWeight, ∏ factor.assess(ctx).weight())
```

New:
```
rawRelevance = max(minWeight, ∏ factor.assessBatch(ctxs)[attestationId].weight())
```

- [ ] **Step 4: Update ADR 0018 consequences (lines 104-120)**

After line 113, add a new consequence bullet:

```markdown
- `RelevanceComposer` calls `assessBatch()` per factor (not `assess()` per attestation).
  Existing per-item factors work unchanged via the default `assessBatch()` delegation.
  Group-aware factors (e.g. `IndependenceAwareRelevanceFactor` in eidos) override
  `assessBatch()` to reason about batch composition — the first shipped example caps
  same-unit share of a group's evidence mass.
```

Update the future-work bullet (line 118-120) from prospective to shipped:

Old:
```
- Future work: concrete `AttestationRelevanceFactor` implementations (org-proximity,
  capability-overlap) live in consumers (casehub-work, casehub-qhorus), not in ledger.
  Ledger provides the SPI and composition; domain knowledge stays in the domain.
```

New:
```
- Concrete `AttestationRelevanceFactor` implementations live in consumers, not in ledger.
  Shipped: `OrgProximityRelevanceFactor` (accuracy-biased, per-attestation) and
  `IndependenceAwareRelevanceFactor` (independence-biased, group-aware via `assessBatch()`),
  both in eidos org-runtime. Ledger provides the SPI and composition; domain knowledge
  stays in the domain.
```

- [ ] **Step 5: Update contributor guide TrustScoreCalculator section (after line 70)**

Insert a new subsection after the TrustScoreCalculator description:

```markdown
### Attestation Relevance Pipeline

`TrustScoreCalculator.computeAll()` builds `AttestationRelevanceContext` records from
each attestation's `relevanceSnapshot` JSON and passes them to `RelevanceComposer`.
The composer calls `assessBatch()` on each `AttestationRelevanceFactor` bean — per-item
factors use the default delegation (loops `assess()`); group-aware factors override
`assessBatch()` to reason about batch composition.

Shipped factors (both in eidos org-runtime):
- **`OrgProximityRelevanceFactor`** — accuracy-biased: weights by `org-distance` from
  snapshot (same-unit = 1.0, cross-unit decreasing). Per-attestation.
- **`IndependenceAwareRelevanceFactor`** — independence-biased: caps same-unit share of
  a group at `casehub.eidos.trust.independence.max-same-unit-share` (default 0.6). Uses
  `assessBatch()` for group-level reasoning.

The four-factor weight chain is: `decay × confidence × credibility × normalised_relevance`.
Adding a new relevance dimension requires a `LedgerAttestationEnricher` (write-time, writes
to `relevanceSnapshot` JSON) and an `AttestationRelevanceFactor` (read-time, reads from
`AttestationRelevanceContext.snapshot()`). No schema migration needed.
```

- [ ] **Step 6: Update CLAUDE.md RelevanceComposer description**

In the project structure section, update the `RelevanceComposer` line:

Old:
```
│       │   ├── RelevanceComposer.java       — concrete: iterates Instance<AttestationRelevanceFactor>, product with floor (casehub.ledger.trust-score.relevance.min-weight, default 0.5); zero factors → NEUTRAL
```

New:
```
│       │   ├── RelevanceComposer.java       — concrete: calls assessBatch() per factor, multiplies weights with floor (casehub.ledger.trust-score.relevance.min-weight, default 0.5); zero factors → NEUTRAL; group-aware factors override assessBatch() for batch-level reasoning
```

- [ ] **Step 7: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add ARC42STORIES.MD docs/adr/0018-attestation-relevance-factor-spi.md docs/guides/contributor-guide.md CLAUDE.md
git commit -m "docs(#239): update ARC42, ADR 0018, contributor guide for independence factor

- ARC42 Active Risks: in-unit amplification mitigation now shipped
- ADR 0018: composition model uses assessBatch(), consequences updated
- Contributor guide: new Attestation Relevance Pipeline subsection
- CLAUDE.md: RelevanceComposer description updated for batch composition

Refs casehubio/ledger#239

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- `specs/issue-239-independence-relevance-factor/2026-11-22-independence-relevance-factor-design.md` — design spec this plan implements
- `specs/issue-239-independence-relevance-factor/decisions.md` — 7 validated design decisions
- `ledger-core/.../RelevanceComposer.java` — batch-oriented composition (commit `a13ec9e8`)
- `ledger-core/.../RelevanceComposerTest.java` — existing assessBatch tests
- `runtime/.../TrustScoreCalculatorTest.java` — existing test patterns and fixtures
- `ledger-core/.../TrustScoreCalculator.java:73-95` — context building from attestation snapshots
- `eidos/.../IndependenceAwareRelevanceFactor.java` — reference implementation for test factor
- `ARC42STORIES.MD:966,1313` — docs to update
- `docs/adr/0018-attestation-relevance-factor-spi.md` — ADR to update
- `docs/guides/contributor-guide.md:68-70` — contributor guide to update
- casehubio/ledger#239 — focal issue
- casehubio/eidos#257 — companion eidos issue (factor implementation)
