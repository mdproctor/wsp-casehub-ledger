# CaseHub Ledger — Session Handover
**Date:** 2026-06-04

## Current State

Both repos on `main`, clean. Pause stack empty. Linear history — zero merge commits.
Both `mdproctor/ledger` and `casehubio/ledger` updated with force-push (`b1ebe44`).

## What Landed This Session

- **#120**: `ARC42STORIES.MD` completed (commit `442c2ae`) then exhaustively reviewed against all 40 blogs, DESIGN.md, DESIGN-capabilities.md, 17 ADRs, and design specs. 11 factual errors corrected + 15 significant omissions added (commit `e24d4fb`). Now accurate.
- **History linearised + squashed**: hidden merge commit `74b34d03` detected; cherry-pick onto feature tip linearised it; `--reapply-cherry-picks` + merge-base fixed the "previously applied" rebase failure; 343 commits → 257, force-pushed to both remotes. CI green, local build green.
- **quarkmind HANDOFF**: ledger#114 blocking dependency cleared (`31283f8` on quarkmind workspace main).
- **Garden**: `GE-20260604-eae751` (linearise merge commit via cherry-pick); REVISE on `GE-20260602-fd2a31` (added `--reapply-cherry-picks` solution).
- **Protocol**: `PP-20260604-9da905` — ARC42STORIES.MD requires all source material (blogs + design docs + ADRs), not just CLAUDE.md. Committed to casehub/garden.
- **Blog**: `2026-06-04-mdp01-reading-the-ledger-backwards.md`, `2026-06-04-mdp02-the-squash-that-needed-surgery.md`
- **Backup branches past 14-day hold**: `backup/pre-squash-main-20260507`, `20260508`, `20260521` — eligible for deletion.

## Immediate Next Step

`#116` — `JpaLedgerEntryRepository.save()` sequenceNumber gap. Self-contained correctness bug, no external deps. Fix: `MAX(sequenceNumber) + 1` per subject with concurrency safety. Unblocks JPA consumers of `OutcomeRecorder`.

## What's Left

- Backup branches `backup/pre-squash-main-20260507`, `20260508`, `20260521` — delete (past 14-day hold) · XS · Low
- `backup/pre-squash-issue-114-lightweight-mode-20260602` — hold until 2026-06-16 · XS · Low
- Missing `chore: branch closed` stamps on shipped issue branches — cosmetic · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #116 | JpaLedgerEntryRepository.save() sequenceNumber gap | M | Med | Self-contained; unblocks JPA consumers of OutcomeRecorder |
| #115 | Incremental per-actor trust recomputation | L | High | Blocked on casehub-engine TrustScoreCache.refreshForActor companion |
| — | QuarkMind: wire OutcomeRecorder into game loop | M | Low | Unblocked by #114; quarkmind-side change |

## References

| Artifact | Where |
|----------|-------|
| ARC42STORIES.MD | `ARC42STORIES.MD` (workspace root) — reviewed 2026-06-04 |
| Blog entries | `blog/2026-06-04-mdp01-*`, `blog/2026-06-04-mdp02-*` |
| Garden entries | `GE-20260604-eae751`, REVISE `GE-20260602-fd2a31` |
| Protocol | `PP-20260604-9da905` (casehub/garden) |
