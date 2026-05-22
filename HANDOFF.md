# CaseHub Ledger — Session Handover
**Date:** 2026-05-22

## Current State

449 tests, BUILD SUCCESS. Both ledger repos on `main`, history clean.
casehub-eidos bootstrapped — new repo on casehubio/eidos and mdproctor/eidos,
workspace on wsp-casehub-eidos. Platform docs updated.

## What Landed This Session

**Housekeeping:** garden#1 (ArchUnit note in reactive-service-build-gating protocol)
and parent#42 (ledger deep-dive + Implementation Protocols table) closed.
ledger#89 (ActorTypeResolver migration) was already done — closed.

**Research → spec → repo:** Six-domain research sweep produced `research/eidos.md`
in the ledger workspace (now also in the eidos workspace). Key findings: LDP's
unverified hints degrade quality below baseline; MAST 36.9% inter-agent
misalignment; SVO over D&D for disposition; two-layer capability (static descriptor
+ dynamic CapabilityHealth probe). Named Eidos over Archetype (Maven collision),
Idos (idOS crypto), Ontos (ontology confusion).

**Platform protocol:** `platform-api-scope.md` added to garden — casehub-platform-api
is for primitives that avoid duplication across peer repos, not a shared types bucket.
PLATFORM.md Step 4 updated. All Eidos types go in `casehub-eidos-api`.

**casehub-eidos bootstrapped:** Maven structure (api/runtime/persistence-memory/
deployment/vocab), all core types in `casehub-eidos-api`, `EidosProcessor` @BuildStep,
publish.yml workflow, parent workflows updated (dashboard, full-stack, incremental).
Bidirectional symlinks and CLAUDE.md cross-references in place. Memory seeded
for eidos workspace.

## Immediate Next Step

Open the eidos workspace and start Phase 1:
```bash
# workspace: /Users/mdproctor/claude/public/casehub/eidos
# project:   /Users/mdproctor/claude/casehub/eidos
```
`work-start` to create a branch + issue, then implement:
- `JpaAgentRegistry` + `JpaReactiveAgentRegistry`
- `CdiVocabularyRegistry`
- `InMemoryAgentRegistry` in persistence-memory/
- SVO, Conscientiousness, CasehubSlot vocab CDI beans in vocab/

## What's Next (ledger)

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

| What | Path |
|------|------|
| Eidos spec | `wksp/research/eidos.md` (or eidos workspace `research/eidos.md`) |
| Latest blog | `blog/2026-05-22-mdp03-giving-agents-a-form.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
