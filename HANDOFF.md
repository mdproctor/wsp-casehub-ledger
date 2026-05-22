# CaseHub Ledger — Session Handover
**Date:** 2026-05-23

## Current State

449 tests, BUILD SUCCESS. Both ledger repos on `main`, casehubio/ledger CI green ✅.
Binaries published to GitHub Packages.

**Eidos handed off** — a separate Claude session is now working in casehub-eidos.
This session is ledger-only.

## What Landed Previously

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up one of the open ledger issues. Most actionable:

- **#97** — Strengthen ArchUnit parity check to verify `Uni<T>` wraps correct return type (S / Low)
- **#91** — Add `persistence-memory/` module — zero-config ephemeral install and in-memory test support (M / Med)
- **#90** — Evaluate `LedgerTraceIdProvider` SPI move to `casehub-platform-api` (S / Med)

Run `work-start` from `main` to branch off the chosen issue.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #97 | Strengthen ArchUnit `Uni<T>` return-type parity check | S | Low | Good warm-up; unblocks confidence in reactive tier |
| #91 | `persistence-memory/` module | M | Med | Useful for eidos too once we confirm scope |
| #90 | `LedgerTraceIdProvider` SPI → platform-api migration | S | Med | Needs platform coherence check first |
| #96 | Code-gen approach for reactive service tier | L | High | Exploratory; no rush |
| #85 | External key distribution (TUF/HSM/PKI) | L | High | Future work |

## References

| What | Path |
|------|------|
| Eidos spec | `wksp/research/eidos.md` (context only — not ledger work) |
| Latest blog | `blog/2026-05-22-mdp03-giving-agents-a-form.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
