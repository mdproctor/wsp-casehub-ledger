---
layout: post
title: "The Sentinel, This Time"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, schema, quarkus, code-review, group-b]
---

Before Group B could start properly, we split `DESIGN.md`. At 503 lines it wasn't unmanageable, but the capability trust work — multi-dimensional scoring, the new query surface, all the things coming in the next few issues — was going to push it past readable. The organizing principle was obvious once named: stable structure stays in `DESIGN.md`, growing algorithms move to `DESIGN-capabilities.md`. It took an afternoon and everything cross-references correctly.

Then we started the actual work: `capabilityTag` on `LedgerAttestation`.

The field scopes each attestation verdict to a capability domain — "security-review", "architecture-review" — or applies it globally. The question was what "globally" means in the schema. NULL is the obvious answer: no tag means all capabilities.

I asked whether NULL smelled. It did.

The last entry covered `scope_key` on `ActorTrustScore` and why NULL was the right answer there: `UNIQUE NULLS NOT DISTINCT` solved the constraint problem, so NULL genuinely meant "no scope." This is a different problem. There's no uniqueness constraint to solve — the issue is query patterns. Every method retrieving global attestations needs `WHERE capabilityTag IS NULL`. IS NULL uses different execution paths than `=` in most databases, behaves differently in JPQL, and compounds wherever you need it. Put enough special cases in your access paths and you've built a system that works differently in the one scenario you didn't test.

The alternative is `CapabilityTag.GLOBAL = "*"`. Every attestation has a tag. Global ones carry `"*"`. The column is `NOT NULL DEFAULT '*'`, the Java field initialises to `GLOBAL`, and queries use `= '*'` throughout — the same operator as any scoped query. Kubernetes RBAC, AWS IAM, Casbin: all use `"*"` for "all resources." Any developer who's written policy rules reads it immediately.

```java
public static final String GLOBAL = "*";
```

```sql
capability_tag  VARCHAR(255)  NOT NULL DEFAULT '*'
```

The irony isn't lost: last entry ended with "the sentinel would have been a lie that survived in the schema indefinitely." Here we chose a sentinel. The difference is what problem you're actually solving. NULLS NOT DISTINCT fixes constraint uniqueness. It doesn't fix IS NULL spreading through your query layer.

## What tests can't see

Nine integration tests covered the new query methods. The final code review found two things the tests couldn't.

The first: `findAttestationsByAttestorIdAndCapabilityTag` was passing the raw attestorId directly to the query. Every other attestorId method in the repository calls `actorIdentityProvider.tokeniseForQuery()` first — the stored value is a token, not the raw identity. Claude caught this. The tests run with tokenisation disabled, so all nine passed. In production with tokenisation active, the method would silently return empty results with no error.

The second: the project had a dead `api/repository/` package — `LedgerEntryRepository`, `ReactiveLedgerEntryRepository`, `ActorTrustScoreRepository` — with zero usages. Every consumer injects from `runtime/repository`. The api copies were months old and accumulating drift. We added three new methods to the runtime SPI; the api package still had the old interface. Claude flagged it by checking usages rather than assuming a class that exists must be live.

Both are the category of correctness issue that lives past the edge of the test suite. The tokenisation bug needs a test running with tokenisation enabled. The dead package needs someone to actually ask "who uses this?" The review is part of the process for a reason.
