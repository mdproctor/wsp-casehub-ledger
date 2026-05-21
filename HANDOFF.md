# CaseHub Ledger — Session Handover
**Date:** 2026-05-21

## Current State

458 tests, BUILD SUCCESS. Both repos on `main`, `.m2` installed.
No active branch. All open issues resolved.

## What Landed This Session

**ledger#95** (closed): Flyway migrations moved from `classpath:db/migration` to
`classpath:db/ledger/migration`. `LedgerProcessor` gains `validateFlywayMigrationLocation`
`@BuildStep` (warns at augmentation when path absent). `FlywayLocationContractTest` pins
the canonical path. 9 example configs updated to both paths. `mvn clean install` required
after path move — plain `install` leaves stale copies in JAR; added second contract test
to catch this.

**qhorus#179** (closed): V1003 `agent_message_ledger_entry` renamed to V2000 (avoids
conflict with ledger's V1003). `quarkus.flyway.qhorus.locations` updated to include
`classpath:db/ledger/migration`. `FlywayMigrationSchemaTest` now scans both paths with
real migrations (stub removed). PLATFORM.md and `ledger-subclass-extension.md` updated
to V2000+ consumer convention.

**work-end skill fix**: Six publish-blog bypass vectors eliminated. 8g folded into 8a
(runs immediately on workspace main after git push, before returning to epic branch).
Close plan always shows Publish blog line. Pre-execution acknowledgement when BLOG_COUNT>0.
Synced to `~/.claude/skills/work-end/SKILL.md` via sync-local.

**Blog backfill**: 30 entries published to mdproctor.github.io — 27 from the rolling
skip bug, 3 from non-main branches. Cross-branch audit script at `/tmp/find_all_blog_entries.py`.

## Immediate Next Step

`gh issue list --repo casehubio/ledger --state open` — check tracker. Top candidates:
#94 (compile-time blocking/reactive parity enforcement, natural follow-on to #93),
#85 (external key distribution TUF/HSM for `AgentKeyProvider`).

## Cross-Module

**Not yet blocking, tracked:**
- `casehub-qhorus#172` — align qhorus with reactive tier pattern · M · Med
- `casehubio/aml#26` — consumer apps add `classpath:db/ledger/migration` to Flyway configs

## What's Left

- `epic-reactive-key-service` workspace branch: no EPIC-CLOSED.md, deletion overdue · XS · Low
- `issue-92-optional-reactive-repo` workspace branch: same · XS · Low
- `main-pre-retro` project branch: stale 4-week-old branch, delete candidate · XS · Low

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
| Latest blog | `blog/2026-05-21-mdp03-the-step-that-kept-skipping.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Blog audit script | `/tmp/find_all_blog_entries.py` |
