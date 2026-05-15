---
layout: post
title: "Quality at the Intersection"
date: 2026-05-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, trust, schema, sql]
---

Knowing an agent is trusted for security reviews tells you they make sound decisions in that domain. It doesn't tell you whether they're thorough. Those are separate questions.

`ActorTrustScore` has had three score types since the capability work shipped: GLOBAL (Bayesian Beta across all decisions), CAPABILITY (Beta scoped to a capability tag), and DIMENSION (decay-weighted average of a quality metric across all capabilities). An attestation on a security review for thoroughness produces both a CAPABILITY row and a DIMENSION row — but the connection is lost. The agent's thoroughness gets averaged across everything they've reviewed, blurring exactly the signal that matters.

The fix is a fourth type: CAPABILITY_DIMENSION. One row per (actor, capability tag, quality dimension).

The schema problem was the first thing to sort out. The existing model used a single `scope_key VARCHAR(255)` — null for GLOBAL, capability tag for CAPABILITY, dimension name for DIMENSION. Adding CAPABILITY_DIMENSION would mean encoding two values into one string: `"security-review:thoroughness"`. That encoding leaks into every query site. LIKE patterns for prefix scans. String splitting to decode. An invisible separator contract between producers and consumers.

I replaced it with two explicit nullable columns:

```
capability_key  VARCHAR(255)   -- null unless capability-scoped
dimension_key   VARCHAR(255)   -- null unless dimension-scoped
```

The four types fall out of which columns are non-null:

| score_type           | capability_key    | dimension_key  |
|----------------------|-------------------|----------------|
| GLOBAL               | null              | null           |
| CAPABILITY           | "security-review" | null           |
| DIMENSION            | null              | "thoroughness" |
| CAPABILITY_DIMENSION | "security-review" | "thoroughness" |

`score_type` stays as a column for indexed queries — without it, every type check needs a two-column IS NULL expression. But a CHECK constraint ties it to the key nullity: the database rejects any combination that doesn't match the state machine. Invalid rows can't be inserted regardless of application bugs or direct SQL writes.

The computation reuses `computeDimensionScore` — the same decay-weighted average already used for DIMENSION scores. A fourth pass in `TrustScoreJob` slots between the dimension pass and the global pass, grouping raw attestations by (capabilityTag, trustDimension) rather than just trustDimension. No new statistical model; the filter is the feature.

The consumer-facing surface in `TrustGateService`:

```java
Optional<Double> qualityScore(String actorId, String capabilityTag, String dimension);
Map<String, Double> qualityScores(String actorId, String capabilityTag);
boolean meetsQualityThreshold(String actorId, String capabilityTag, String dimension, double min);
```

When routing a high-stakes review, a consumer can ask whether an agent meets a quality floor specifically for that type of work — not just whether they're trusted to do it.

Code review surfaced two things. The V1005 migration was an ALTER script on top of V1001, which contradicts a project convention: with no deployed instances, schema changes go directly into the base migration file. V1005 was deleted and V1001 rewritten with the final two-column schema. The other was a Javadoc contradiction — CAPABILITY_DIMENSION appeared in both the "binary scores use Bayesian Beta" and "continuous scores use decay-weighted average" groups. It uses decay-weighted average; the first sentence was wrong.

ADRs 0009 and 0010 record the two decisions: why continuous quality scores use decay-weighted average rather than Beta, and why two explicit columns beat a composite string.
