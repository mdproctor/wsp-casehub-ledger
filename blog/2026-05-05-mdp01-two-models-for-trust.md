---
layout: post
title: "Two Models for Trust"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, bayesian, quarkus, code-review]
---

The capability scores added in the previous phase use the Bayesian Beta model. A binary verdict — SOUND or FLAGGED — increments α or β, decayed by age and scaled by confidence.

Dimension scores don't fit that model. A continuous quality measurement like "thoroughness: 0.78" isn't a positive or negative signal — it's a magnitude. Running it through a Beta accumulator produces an α/β ratio that doesn't mean anything useful.

I needed a different computation path. Claude and I worked out a decay-weighted average: for each (actor, dimension) pair, sum `weight × confidence × dimensionScore` across all attestations and divide by the total weight. Same exponential decay as the Beta model, same confidence scaling, same per-attestation clamping — but averaging rather than accumulating:

```java
final double weight = decayFunction.weight(ageInDays, AttestationVerdict.SOUND)
        * Math.max(0.0, Math.min(1.0, a.confidence));
weightedSum += weight * a.dimensionScore;
totalWeight += weight;
```

The `AttestationVerdict.SOUND` argument is the interesting choice. The decay function applies a slower-decay multiplier to FLAGGED/CHALLENGED attestations — negative evidence persists longer. That asymmetry makes sense for binary verdicts but not for continuous scores, which have no polarity. Passing SOUND forces pure time-based decay and drops the valence logic.

Both models share the same `actor_trust_score` table. DIMENSION rows use `scope_key = dimensionName` the same way CAPABILITY rows use `scope_key = capabilityTag`. The `alpha_value` and `beta_value` columns hold 0.0 for dimension rows — not meaningful for an averaging model, but the schema doesn't distinguish.

## Three catches

The first implementation of `computeDimensionScore` omitted confidence weighting — pure time-based decay, no confidence scaling. Claude caught the asymmetry: `compute()` multiplies by confidence; the dimension path did not. Whether confidence means the same thing for a continuous score as for a binary verdict is a real design question. I think it does — confidence represents how certain the attestor is about the measurement, orthogonal to the magnitude. One-line fix.

Two more in the final review. The binary positive/negative audit counter used `> 0.5` for positive and `<= 0.5` for negative, so an exactly-neutral score of 0.5 was filed as negative. These counters don't affect `trustScore` — the score comes from the weighted average — but the audit record misrepresents neutral attestations. Fixed to `>= 0.5` / `< 0.5`.

The last one: the `trust-score-test` Quarkus profile enabled the feature flag but didn't set `quarkus.scheduler.enabled=false`. Tests call `runComputation()` directly — if the scheduler fires concurrently, both write to the same H2 tables in the same window. The routing test profiles in the same file had the guard with a comment explaining why; this one was missing it. The 24-hour interval makes the race rare, which is the worst kind of omission — the kind that produces intermittent failures nobody can reproduce.

303 tests.
