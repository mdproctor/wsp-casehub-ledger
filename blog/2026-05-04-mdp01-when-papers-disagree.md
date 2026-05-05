---
layout: post
title: "When the Papers Disagree"
date: 2026-05-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, bayesian, spi, quarkus, code-review]
---

Before starting #61 I wanted to know what the literature says about global trust score aggregation. The question seemed simple: when an agent has both global and capability-specific attestations, does the global score use all of them or only the ones tagged as explicitly cross-capability? I was about to just pick one and move on.

The papers didn't agree.

Wang & Vassileva (2003) treat global trust as the root node of a Bayesian network, computed from all interactions. Capability-specific scores are derived conditional views on top of it. Their explicit quote: "we do not treat the differentiated trusts as compositional — the relationship between different aspects of an agent is not just compositional, but complex and correlative." Global first, capabilities derived.

Fan et al. (2015) disagree. Their overall trust is a weighted average of dimension-specific scores — `TV = Σ wᵢ × trust_dimension_i`. Global is derived from capabilities, not the other way around.

Jøsang's original Beta Reputation System doesn't address the question at all. Single context only.

Three legitimate positions, backed by peer-reviewed papers. The right answer depends on what "global trust" means semantically — and that's a deployment question, not an algorithm question. I wasn't going to resolve it by picking one.

## The SPI answer

The right move was to make it pluggable. We built `GlobalScoreStrategy` — a CDI SPI with two methods: `selectAttestations(all)` filters which attestations feed the global Beta model, and `derive(capabilityScores, all)` optionally overrides the Beta result after capability scores are computed.

Three implementations ship with the extension:

```java
// Option B (default) — Wang & Vassileva root-node model
@ApplicationScoped @DefaultBean
class AllAttestationsGlobalStrategy implements GlobalScoreStrategy {
    public List<LedgerAttestation> selectAttestations(List<LedgerAttestation> all) {
        return all;  // all attestations feed the global Beta
    }
}

// Option A — explicit-global semantic
@ApplicationScoped @Alternative
class ExplicitGlobalAttestationsStrategy implements GlobalScoreStrategy {
    public List<LedgerAttestation> selectAttestations(List<LedgerAttestation> all) {
        return all.stream().filter(a -> CapabilityTag.GLOBAL.equals(a.capabilityTag))
                .collect(Collectors.toList());
    }
}

// Option C — Fan et al. frequency-weighted combination
@ApplicationScoped @Alternative
class FrequencyWeightedGlobalStrategy implements GlobalScoreStrategy { ... }
```

Option B is the default. My rationale: simplest, no dependency on capability scores being computed first, consistent with Wang & Vassileva's root-node view. Deployments with real data can switch to Option C via `quarkus.arc.selected-alternatives` — no code change, no rebuild.

## The ones the reviewers caught

The implementation of `TrustScoreJob`'s capability pass had an O(N×M) problem I didn't notice. For each distinct capability tag, the original code re-streamed the full `attestationsByEntry` map and filtered it — one full stream per capability tag. With N tags and M entries, that's O(N×M) work.

Claude's quality reviewer caught it. The fix was obvious once named: nested `Collectors.groupingBy` produces `Map<String, Map<UUID, List<LedgerAttestation>>>` in a single O(M) pass.

```java
// O(M) — one pass, directly usable per capability tag
Map<String, Map<UUID, List<LedgerAttestation>>> byCapabilityAndEntry =
    actorAttestations.stream()
        .filter(a -> !CapabilityTag.GLOBAL.equals(a.capabilityTag))
        .collect(Collectors.groupingBy(
            a -> a.capabilityTag,
            Collectors.groupingBy(a -> a.ledgerEntryId)));
```

The final code review also surfaced a subtler issue: `FrequencyWeightedGlobalStrategy.derive()` computes `trustScore` as a weighted average but computes `alpha`/`beta` on a different formula path. They don't satisfy `alpha/(alpha+beta) == trustScore`. The scalar score is correct — everything that routes work uses only `trustScore`. But alpha/beta are persisted to the database and could mislead something that tried to use them as Beta posterior parameters. We documented this as a known limitation in the class Javadoc rather than papering over it.

272 tests. #62 (multi-dimensional trust infrastructure) is next.
