# 0016 — EigenTrust applicability: single-attestor and small-graph deployments

Date: 2026-06-02
Status: Accepted

## Context and Problem Statement

EigenTrust (Kamvar et al. 2003) computes transitive global trust scores by treating
the attestation history as a directed graph and running power iteration to convergence.
`casehub-ledger` supports EigenTrust via `casehub.ledger.trust-score.eigentrust.enabled`.
Consumers deploying the ledger for game-AI or other outcome-tracking use cases (single
authoritative attestor, small fixed agent set) need guidance on when EigenTrust is
appropriate and when it produces degenerate or convergence-unsafe results.

## Decision Drivers

* EigenTrust is meaningless when all attestations originate from a single attestor —
  there is no transitive peer-to-peer trust graph to propagate
* Small graphs (fewer than ~10 distinct attestors) with a pre-trusted fallback are
  susceptible to 3-cycle non-convergence (GE-20260421-09d636)
* Bayesian Beta direct scores capture outcome history correctly without a peer graph
* ledger consumers must not silently activate EigenTrust and receive degenerate scores

## Considered Options

* **Option A** — Enable EigenTrust unconditionally; document it as mandatory
* **Option B** — Leave EigenTrust opt-in (current default=false), document applicability criteria
* **Option C** — Remove EigenTrust and rely solely on Bayesian Beta

## Decision Outcome

Chosen option: **Option B**, because EigenTrust is genuinely valuable in multi-attestor
peer-review deployments (compliance audit, multi-agent research networks) but actively
harmful in single-attestor deployments. The existing opt-in default is correct; this ADR
adds the applicability criteria that were previously undocumented.

### Positive Consequences

* Consumers can make an informed choice based on documented criteria
* EigenTrust remains available for the multi-attestor cases where it is correct
* No code changes — this decision captures existing design intent

### Negative Consequences / Tradeoffs

* Consumers who enable EigenTrust without reading the criteria will still produce bad results;
  a build-time or startup warning would be better (tracked: casehubio/ledger#114)

## Applicability Criteria

Enable EigenTrust (`casehub.ledger.trust-score.eigentrust.enabled=true`) only when **all**
of the following hold:

1. **Multiple independent attestors exist** — at least 3–5 actors attest to each other's
   decisions. A single system actor attesting all plugins produces a star graph; power
   iteration on a star graph is equivalent to direct trust from the system actor and adds
   no value.
2. **The agent graph is sufficiently large** — at least ~10 distinct actors. Smaller graphs
   risk the 3-cycle non-convergence failure when the pre-trusted fallback is used for actors
   with no attestation history (GE-20260421-09d636).
3. **Transitive trust is meaningful** — you care about "do peers trust this actor" not just
   "did outcomes from this actor match expectations". Outcome-tracking deployments (game AI,
   A/B policy evaluation) should use Bayesian Beta direct scores only.

When the criteria are not met, leave EigenTrust disabled and rely on
`casehub.ledger.trust-score.eigentrust.enabled=false` (the default).

## Pros and Cons of the Options

### Option A — Enable unconditionally

* ✅ Simple for operators — no decision required
* ❌ Produces meaningless scores in star-graph deployments
* ❌ 3-cycle non-convergence in small graphs is silent — no error, wrong result

### Option B — Opt-in with documented criteria (chosen)

* ✅ Correct by default — wrong for most initial deployments
* ✅ Preserves value for multi-attestor deployments
* ❌ Relies on consumers reading documentation

### Option C — Remove EigenTrust

* ✅ Eliminates incorrect usage entirely
* ❌ Removes genuine value for large peer-review deployments
* ❌ Irreversible — removing a feature is a breaking change

## Links

* ADR 0003 — Bayesian Beta trust model
* GE-20260421-09d636 — EigenTrust 3-cycle non-convergence in small graphs
* casehubio/ledger#114 — lightweight outcome-tracking mode (QuarkMind)
* Kamvar et al. (2003) — "The EigenTrust Algorithm for Reputation Management in P2P Networks"
