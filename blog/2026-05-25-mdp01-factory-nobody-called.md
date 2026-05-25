---
layout: post
title: "The Factory Nobody Called"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [ledger]
tags: [testing, dead-code, cleanup]
---

An untracked file in `persistence-memory/src/test/` caught my attention at
the start of the session. `TestEntryFactory.java`. Not in git, not committed,
no history at all — `git log --all -- <path>` returned nothing.

That's a warning sign. Either it was forgotten in the middle of some work, or
it was deliberately created and then left behind. We checked every reference
across the test tree: nothing. No test imported it, no test injected it.

The class itself explained the intent. An `@ApplicationScoped` CDI bean designed
to build `MemoryTestEntry` fixtures from inside the Quarkus application
classloader — bypassing the bytecode enhancement boundary that makes direct
field access from test classes unreliable:

```java
@ApplicationScoped
public class TestEntryFactory {
    public MemoryTestEntry entry(UUID subjectId, LedgerEntryType type) {
        final MemoryTestEntry e = new MemoryTestEntry();
        e.subjectId = subjectId;
        // ...
        return e;
    }
}
```

The concern it's solving is real. Quarkus enhances `@Entity` classes and puts
them in a separate classloader; tests that try to assign fields directly outside
that classloader get subtle failures. The factory solves it by running inside the
application CL.

But `MemoryTestEntry.of()` already solves it the same way — a static factory on
the entity subclass itself, which lives in the right classloader by definition.
Every test was already calling that directly, with a thin private `entry()` helper
for brevity. The CDI factory arrived at a problem that the tests had already
organised around.

Dead from the start.

The rest of the cleanup was mechanical: four squash-plan docs left in `docs/`
from prior sessions, a `wksp` symlink that wasn't in `.gitignore`. Both now
gitignored — `wksp` is a local dev convenience that belongs on the machine, not
in the repo.

497 tests green throughout.
