# Session Handoff — 2026-06-18

## Branch closed: issue-151-on-conflict-do-update

treblree filed a detailed issue claiming ON CONFLICT DO NOTHING was masking a
concurrent data race in `LedgerSequenceAllocator` rather than fixing it.
Exhaustive execution trace proved the implementation was correct — H2 MODE=PostgreSQL
blocks TX2 at the INSERT step (index-entry lock), so DO NOTHING correctly resolves
the expected post-commit conflict and the UPDATE row lock holds the Merkle
Serialization Invariant.

The investigation surfaced a real improvement: H2 2.4.240 in MODE=PostgreSQL rejects
`ON CONFLICT (col, col) DO UPDATE` even though it accepts `ON CONFLICT DO NOTHING`.
This forced a three-way `Dialect` enum (POSTGRESQL / H2_PG_MODE / H2_STANDARD).
Real PostgreSQL now gets a single-statement `INSERT … ON CONFLICT DO UPDATE` upsert;
H2 MODE=PostgreSQL retains DO NOTHING + UPDATE; H2 standard gets a full MERGE with
both WHEN MATCHED and WHEN NOT MATCHED clauses.

Detailed technical analysis posted to GitHub issue #151. Two garden entries submitted
(H2 MERGE locking; revise of existing H2 ON CONFLICT entry). All 868 tests pass.

## Current state

- `casehubio/ledger` main: `02f9ec6` — single squashed commit, pushed to both fork
  and blessed repo
- 868 runtime + persistence-memory tests pass (all modules, all tests including Docker)
- Issue #151 closed
- ARC42STORIES.MD, DESIGN.md, CLAUDE.md all updated

## Immediate Next Step

Pick from the backlog — no open obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | blocker #85 closed — ready |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | blocker #85 closed — ready |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count ≥ 5 |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | wait for casehub-ops consumer |
