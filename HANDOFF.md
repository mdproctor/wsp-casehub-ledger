# CaseHub Ledger — Session Handover
**Date:** 2026-05-22

## Current State

447 tests, BUILD SUCCESS. Both repos on `main`, `.m2` installed.
No active branch. `casehub-platform-api:0.2-SNAPSHOT` now a dependency of `casehub-ledger-api`.

## What Landed This Session

**ledger#88** (closed): `ActorType` and `ActorTypeResolver` migrated from
`casehub-ledger-api` to `casehub-platform-api`. All imports updated across
57 runtime + test + example files. `ActorTypeResolverTest` deleted (test
coverage now lives in platform). `CurrentPrincipal.actorType()` unblocked.

Gotcha: files using `ActorType` via same-package resolution (no import)
are invisible to import-search tools — only the build finds them.
Pattern to catch them pre-build: `grep -r "TypeName" src/ --include="*.java" | grep -v "import"`

## Immediate Next Step

`gh issue list --repo casehubio/ledger --state open` — check tracker.
Top candidates: #94 (compile-time blocking/reactive parity enforcement),
#85 (external key distribution TUF/HSM for `AgentKeyProvider`).

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #94 | Enforce blocking/reactive parity at compile time | S | Med | Natural follow-on from #93 |
| #85 | External key distribution — TUF/HSM/PKI for `AgentKeyProvider` | M | Med | — |
| #81 | Agent DID/VC identity binding | M | High | ADR before code |
| #84 | Post-quantum migration path | M | High | ADR before code |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-22-mdp01-actor-type-finds-its-home.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
