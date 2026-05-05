---
layout: post
title: "When the Paper Is Wrong"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, eigentrust, trust-scoring, algorithms]
---

EigenTrust transitivity rated ★★☆☆☆ for enterprise applicability in the capabilities guide —
honest about where procurement conversations are in 2026. But that's the wrong frame for why
we built it. Large-scale AI agent meshes will need transitive trust. The question isn't whether,
it's when. Building the foundation correctly now is cheaper than rearchitecting it once the
mesh exists. This feature is a bet on where agentic AI is going, not a response to where it is.

## The Research Questions Had Obvious Answers

Issue #26 listed three questions before any code: is power iteration feasible at our scale?
How do we seed the trust matrix from Beta posteriors? What's the pre-trusted peer set?

The first question is trivial — power iteration on a 100×100 matrix converges in ~30 iterations.
The agent meshes we're dealing with have tens of actors, not millions.

The second question drove the design. C[i][j] is the trust actor i places in actor j —
derived from attestation data. Specifically: the positive attestations i has made about j's
decisions, normalised by all positive attestations from i. Negative attestations subtract from the
raw signal before normalisation. The pre-trusted set seeds the distribution that anchors the
dampening step.

## The Paper Has a Bug

Before writing a line of code, I traced the algorithm by hand for a 3-actor example: A trusts B,
B trusts C, C has no attestation history. The paper's recommendation for actors with no positive
attestations is to use the pre-trusted distribution as their matrix row. So C's row becomes
[1, 0, 0] — all weight back to pre-trusted A.

That creates a 3-cycle: A→B→C→A. Power iteration oscillates with period 3 instead of converging.

I worked through eight iterations. The values kept shifting — [0.76, 0.13, 0.11], then
[0.30, 0.53, 0.18], then [0.24, 0.37, 0.39]. No convergence trend. The Stanford paper targets
large P2P networks where an exact 3-cycle is statistically improbable. For a ledger with a
handful of agents, it's the default topology.

The fix: uniform distribution (1/n) for actors with no positive attestations — the standard
dangling-node treatment. Reserve the pre-trusted distribution exclusively for the dampening step:

```
t = (1-α) * Cᵀ * t + α * p
```

Verified by tracing a 4-actor chain: converges in six iterations to A>B>C>D, with C>D
demonstrating transitivity.

## Two Scores, Not One

`EigenTrustComputer` is pure Java, no CDI, same pattern as `TrustScoreComputer`. It takes all
attestations plus a map from entry ID to decision-making actor, builds the trust matrix, runs
power iteration, and returns eigenvector components in [0.0, 1.0] summing to 1.0 across all actors.

`TrustScoreJob` runs the eigentrust pass as a second phase after the Beta pass. A new
`global_trust_score` column lands on `actor_trust_score` — V1001 updated in place. The two scores
aren't comparable directly: the Beta score measures reliability from direct attestations; the
EigenTrust score is a PageRank-style share of global trust.

Three new config keys: `eigentrust-enabled` (default false), `eigentrust-alpha` (default 0.15),
`pre-trusted-actors` (list, default empty). Eight unit tests covering transitivity, negative
attestations, and alpha sensitivity. 167 tests total, all passing.
