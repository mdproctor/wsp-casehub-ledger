---
layout: post
title: "The Proxy and the Bean"
date: 2026-05-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, cdi, interceptor, testing]
---

Something came up twice this session in slightly different forms: the gap between
a CDI proxy reference and the bean it actually delegates to.

The first was a design choice. `@ProvenanceCapture` is an interceptor binding that
auto-attaches `ProvenanceSupplement` to any `LedgerEntry` persisted within the
annotated method. The interceptor sets a context object before the call; a
`LedgerEntryEnricher` reads it at `@PrePersist` time.

The natural carrier for that context is `@RequestScoped`. I used `@ApplicationScoped`
with a `static ThreadLocal<Deque<SourceState>>` instead. The reason: `@RequestScoped`
requires an active HTTP request context. Scheduled jobs don't have one. `@QuarkusTest`
without HTTP doesn't have one. The failure would be a runtime `ContextNotActiveException` —
not a compile error, not something you catch until integration tests run in the wrong
configuration.

A static `ThreadLocal` sidesteps the scope dependency entirely. Thread-local storage is
effectively request-scoped for most workloads, and the `static` modifier means the field
is shared across the proxy/bean split — no proxy field access trap:

```java
private static final ThreadLocal<Deque<SourceState>> STACK =
        ThreadLocal.withInitial(ArrayDeque::new);
```

`STACK.remove()` when the deque empties prevents classloader leaks on hot reload.

The second time was a test failure. `LedgerHealthJob` has a configurable reconciliation
check that queries registered `LedgerReconciliationSource` beans. In the integration test,
we defined a `@ApplicationScoped` inner class with an `active` flag. The test set
`reconciliationSource.active = true` directly. The reconciliation never fired.

Arc subclass proxies intercept method calls by overriding them. Field access bypasses
the proxy — it goes to the proxy class's own field.

So `reconciliationSource.active = true` wrote to the proxy's field;
`source.isActive()` read the bean instance's field, which remained `false`. No error,
no warning. The fix was static fields on the inner test bean — static fields are shared,
visible from both the proxy reference and the actual instance. Claude identified the
mismatch during the failure investigation.

## Two things the review caught

The compliance report service came back from review with two findings.

The first was a bad API contract. `reportForActor()` and `reportForSubject()` both
accepted a `ReportFormat` argument. Neither used it — the returned `ComplianceReport`
was always raw data, and callers still had to invoke `report.format(ReportFormat.CSV)`
themselves. The signature claimed to apply the format; it didn't. Claude flagged this
as Critical. We removed the parameter: callers format the report themselves, which is
the right model for a CDI service.

The second was smaller. `ic.getTarget().getClass()` returns the Arc proxy class, which
doesn't carry annotations. A class-level `@ProvenanceCapture` annotation would be
silently ignored — `sourceEntityType` and `sourceEntitySystem` would come back empty.
`ic.getMethod().getDeclaringClass()` is the correct call. This only affects type-level
annotations, and there were no tests exercising that path, which is why neither of us
caught it before review.

345 tests.
