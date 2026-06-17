# Session Handoff — 2026-06-17

## Branch closed: issue-146-key-rotation-tenancy

`KeyRotationRepository` had the same structural tenancy gap as #144/#145.
Two methods, two different answers:

- `findByActorId(actorId, tenancyId)` — now tenant-scoped. Each tenant sees
  only its own rotation history.
- `findCompromisedByActorIdAndKeyRef(actorId, keyRef)` — stays cross-tenant.
  A compromised key is a global security signal; scoping per-tenant would let
  a security incident in one deployment stay invisible to others sharing the
  same key pair. The intent was already there — `AgentSignatureVerificationService`
  had always called `compromisedWindows` without tenancyId.

Protocol PP-20260616-7d4171 updated with the security-query exception clause.
All tests green. Pushed to casehubio/ledger main as a single squashed commit.

Also fixed CI: `consumer-compat-test` was failing deploy because the fix commit
(`28bca4d`) was on the fork but not on upstream. `git ls-remote upstream main`
showed the gap; `git push upstream main` resolved it.

## Current state

- `casehubio/ledger` main: `9184e8a` — 1 squashed commit ahead of where the session started
- All tests pass (BUILD SUCCESS, 3m 41s); PostgreSQL tests require Docker
- Issue #146 closed on GitHub
- Blog entry published: `2026-06-17-mdp01-compromise-crosses-tenants.md`

## What's Next

No open issues identified. Next work is discretionary — pick from the backlog.
