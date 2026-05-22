# CaseHub Ledger — Session Handover
**Date:** 2026-05-22

## Current State

449 tests, BUILD SUCCESS. Both ledger repos on `main`, casehubio/ledger CI green ✅.
Binaries published to GitHub Packages. casehub-eidos bootstrapped on casehubio/eidos
and mdproctor/eidos, workspace on wsp-casehub-eidos.

## What Landed This Session

**Housekeeping:** garden#1, parent#42, ledger#89 closed.

**Research → spec → repo:** Six-domain research sweep produced `research/eidos.md`.
Key findings: LDP unverified hints degrade quality; MAST 36.9% inter-agent misalignment;
SVO over D&D for disposition; two-layer capability (static descriptor + dynamic
CapabilityHealth probe). Named Eidos over Archetype (Maven collision), Idos (crypto),
Ontos (ontology confusion).

**Platform protocol:** `platform-api-scope.md` added to garden. PLATFORM.md updated.

**casehub-eidos bootstrapped:** Maven structure, all core types in `casehub-eidos-api`,
`EidosProcessor` @BuildStep, publish.yml, parent workflows updated. Bidirectional
symlinks, CLAUDE.md cross-references, memory seeded for eidos workspace.

**CI fixes:**
- `casehub-platform` publish.yml: now dispatches to ledger + connectors after publish
- `casehub-ledger` pom.xml: added `<repositories>` section for casehub-platform-api
  resolution (introduced as a dep by #88 but no repo declared — matched qhorus pattern)

## Immediate Next Step

Open the eidos workspace and start Phase 1:
```
workspace: /Users/mdproctor/claude/public/casehub/eidos
project:   /Users/mdproctor/claude/casehub/eidos
```
`work-start` → branch + issue → implement:
- `JpaAgentRegistry` + `JpaReactiveAgentRegistry`
- `CdiVocabularyRegistry`
- `InMemoryAgentRegistry` in persistence-memory/
- SVO, Conscientiousness, CasehubSlot vocab CDI beans in vocab/

## References

| What | Path |
|------|------|
| Eidos spec | `wksp/research/eidos.md` |
| Latest blog | `blog/2026-05-22-mdp03-giving-agents-a-form.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
