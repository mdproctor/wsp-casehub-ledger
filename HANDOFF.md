# Session Handoff — 2026-06-22

## Branch closed: issue-157-flagged-capability-scores

Fixed #157 — FLAGGED attestations on the same entry as a SOUND attestation were masked by WEIGHTED_MAJORITY aggregation in the capability scoring pass. Root cause: `TrustScoreCalculator.computeAll()` used pre-aggregated attestations for the capability pass, collapsing contradicting verdicts into a near-zero-weight consensus. Fix: use raw attestations (matching the dimension pass). Garden entry GE-20260625-5287ac captures the gotcha.

## Current state

- `casehubio/ledger` main: `c45f126` — pushed to origin + upstream
- All 5 modules BUILD SUCCESS
- Issue #157 closed
- Idea logged: hybrid attestation weighting for capability scoring (multi-attestor N× weight trade-off)

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm (first clarifying question asked: per-hash vs lineage identity model). Resume with `/work`.

## Immediate Next Step

Resume #137 brainstorm, or pick from the backlog.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | paused — brainstorm in progress |
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | blocker #85 closed — ready |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | blocker #85 closed — ready |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count ≥ 5 |
