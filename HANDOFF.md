# Session Handoff — 2026-07-07

## Diagnostic session — no code changes

Investigated GitHub Packages API reporting `casehub-ledger:0.2-SNAPSHOT` `updated_at`
as May 1. Root cause: the API shows version *creation* date, not last artifact upload.
CI logs confirm `mvn deploy` uploads fresh artifacts on every push to main. No action
needed — publishing is working correctly.

## Current state

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Paused branch

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*
