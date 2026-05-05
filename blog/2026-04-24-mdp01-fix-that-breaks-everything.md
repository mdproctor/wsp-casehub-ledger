---
layout: post
title: "The Fix That Would Have Broken Everything"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [cdi, quarkus, org-structure]
---

Claude added `@PersistenceUnit("qhorus")` to six runtime beans in casehub-ledger
without being asked. The commit message was plausible: *"Qualifies EntityManager
injections so beans resolve correctly when only the qhorus datasource is configured."*

I caught it reviewing `git log` the next session. The commit was gone within minutes.

## Why it was wrong

`casehub-ledger` is a generic extension. Hardcoding `"qhorus"` in
`JpaLedgerEntryRepository`, `LedgerErasureService`, `TrustScoreJob`, and the
rest means every consumer that isn't Qhorus fails with the mirror error —
`"Unsatisfied dependency for type EntityManager and qualifiers
[@PersistenceUnit("qhorus")]"`. CaseHub would break. casehub-work would
break. Any future consumer with a different datasource name would break on startup.

The error Claude was fixing was real: Claudony configures only a named `qhorus`
datasource with no default, and `@Default EntityManager` is absent from the CDI
context when that's the case. The symptom pointed at a missing annotation. The
wrong response was to add the consumer's annotation to the extension.

## The right fix

The extension doesn't know — and shouldn't know — what datasource its consumers
configure. The right design is a config key that lets each consumer tell the
extension which persistence unit to use:

```properties
# Only needed when the app has no default datasource
quarkus.ledger.datasource=qhorus
```

We built this with a `@LedgerPersistenceUnit` CDI qualifier and a producer that
reads the config and selects the right `EntityManager` via
`Instance<EntityManager>.select()` at startup:

```java
@Produces @LedgerPersistenceUnit
public EntityManager produce(@Any Instance<EntityManager> instance) {
    String pu = config.datasource().orElse("").trim();
    if (pu.isBlank()) {
        return instance.select(Default.Literal.INSTANCE).get();
    }
    return instance.select(new PersistenceUnitLiteral(pu)).get();
}
```

All eight injection points in the runtime module now use `@LedgerPersistenceUnit`.
The fix is IntelliJ-stable — it's a real CDI annotation the IDE understands, not
a string-valued annotation that import optimisers strip.

## Moving to casehubio

The repo transferred to the `casehubio` GitHub organisation. I'm now a shared
owner there alongside the casehub-engine and the rest of the ecosystem. The
artifact coordinates stay the same — only the GitHub URL changed.

## The shared BOM

The transfer also pushed the BOM question into focus. As more repos move to
`casehubio`, they need a shared way to manage dependency versions. The pattern
from Quarkiverse, SmallRye, and Apache is a `<org-name>-parent` pom that serves
as both parent and BOM. That repo is now at `casehubio/parent`.

I chose BOM-import rather than inheritance for the initial wire-up. The projects
in the ecosystem have heterogeneous parents and a shared inheritance chain would
force awkward multi-level structures. Importing the BOM in
`<dependencyManagement>` avoids that without giving up centralised version
management.
