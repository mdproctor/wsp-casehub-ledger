---
layout: post
title: "The log that wouldn't be caught"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, testing, logging, jboss-logmanager, native-image, in-memory]
---

Six S-scale issues. The kind of work that sounds small until you're doing it — one
test gap here, one undocumented method there, one missing `@BuildStep`. Each takes
thirty seconds to describe and thirty minutes to get right.

The one that stuck was #119.

`EigenTrustStartupValidator` observes `StartupEvent` and logs a WARNING when EigenTrust
is enabled with fewer than three pre-trusted actors. The unit test covers the logic
directly — `shouldWarn(true, 0)` returns true, case closed. But the spec required an
integration test verifying that the WARNING actually fires in a real CDI context, with
real config injection. We needed to catch the log output.

First attempt: install a `java.util.logging.Handler` in `@QuarkusTestResource.start()`
before the app starts. The handler receives nothing. The warning is visible in console
output — it's being emitted — but the handler's capture list is empty after every test.

Second attempt: `org.jboss.logmanager.Logger.getLogger(name).addHandler(handler)`,
calling JBoss LogManager's own API directly. Same result. Empty. The warning prints.
The handler is silent.

The fix looked obvious from here: JBoss LogManager's handler chain is re-initialised
during Quarkus bootstrap, after `QuarkusTestResourceLifecycleManager.start()` returns.
Whatever handler you install pre-bootstrap gets attached to a logger instance that gets
discarded when the LogManager rebuilds its tree. By the time `StartupEvent` fires, the
chain is empty again. No error. No indication anything went wrong.

The workaround doesn't try to intercept the log at all. Instead, we verify CDI wiring
(the bean is injectable), verify config resolution (eigentrust.enabled=true, pre-trusted
actors empty), then note that `shouldWarn(true, 0)` is already proven true by the unit
test. The three parts compose. The integration test proves the CDI path; the unit test
proves the logic; together they establish that the WARNING fires.

It's a slightly odd shape for a test — passing by composition rather than by direct
assertion — but the alternative is a test that asserts nothing while looking like it
does, which is worse.

The other five went more cleanly. `NativeImageResourcePatternsBuildItem.builder().includeGlob()`
converts globs to regex internally — `getIncludePatterns()` returns `[^/]*\.sql`, not
`*.sql`. The first test assertion failed on exactly this, which turned out to be worth
a garden entry (GE-20260603-83883c). The fix was a behavioral assertion: does the
pattern match `V1000__ledger_base_schema.sql`? If yes, correct.

`InMemoryAgentSigner` went in cleanly — same `@Alternative @Priority(1)` pattern as
every other in-memory stub, backed by a `ConcurrentHashMap<String, KeyPair>`. The
`clear()` method on `InMemoryLedgerEntryRepository` already existed but was undocumented
as a production lifecycle hook — just Javadoc and two tests to make explicit what had
been implicit.

Two peer-repo issues filed (platform#62 for ScimActorDIDProvider fixes, qhorus#241
for a redundant glob cleanup). Six issues closed. Pause stack cleared of a leftover
from the previous session. The project is clean.
