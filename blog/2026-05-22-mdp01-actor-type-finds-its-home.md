---
layout: post
title: "ActorType Finds Its Home"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [ledger]
tags: [java, maven, refactoring]
---

`ActorType` (HUMAN/AGENT/SYSTEM) has lived in `casehub-ledger-api` since the beginning. It's the right concept in the wrong address — identity classification belongs in `casehub-platform-api`, not in an audit ledger library.

`casehub-platform-api:0.2-SNAPSHOT` is installed. Claude and I ran the migration.

The approach was a Python script: batch-replace every `import io.casehub.ledger.api.model.ActorType;` across the project. 57 files, one pass. That worked cleanly. The problem was with files that had no import statement at all.

`LedgerEntry`, `LedgerAttestation`, and `ActorTrustScore` all declare `ActorType` fields. They were originally in the same package as the enum, so they never needed an import — Java resolves the type implicitly. The script found nothing to match and quietly moved on.

The build caught all three. Adding explicit imports was the fix. But when migrating a type across Maven artifact boundaries, searching by import statement isn't the whole picture — files using the type via same-package resolution have no import to match. A bare name search finds them before the compiler does:

```bash
grep -r "TypeName" src/ --include="*.java" | grep -v "import"
```

`ActorTypeResolverTest` was also deleted — test coverage belongs where the type lives, and `casehub-platform-api` already has it.

`CurrentPrincipal` in `casehub-platform-api` had been carrying a TODO comment since the interface was first written: "add `ActorType actorType()` once ActorType migrates from casehub-ledger-api." That comment is gone now too. 447 tests passing.
