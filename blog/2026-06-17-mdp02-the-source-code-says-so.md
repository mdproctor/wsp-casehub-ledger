---
layout: post
title: "The Source Code Says So"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [h2, api-design, trust-scoring]
---

Four issues landed today on a single branch, and the most interesting thing about three of them is what I didn't know going in.

**The H2 concurrent race was straightforward to fix, once I understood why it existed.** The `MERGE INTO` statement in `LedgerSequenceAllocator` had been blamed on H2 not supporting `ON CONFLICT` — and that was true when the code was written. H2 2.0 shipped `ON CONFLICT DO NOTHING` support in 2022. So the obvious fix was to replace `MERGE` with the two-statement pattern already working on PostgreSQL: `INSERT ... ON CONFLICT DO NOTHING` followed by `UPDATE`. That's it. Both branches go away. One implementation.

Except it didn't compile. H2 threw a syntax error right at the `(` in `ON CONFLICT (subject_id, tenancy_id) DO NOTHING`.

I opened H2's `Parser.java` from the sources JAR in `~/.m2/repository` and found this:

```java
if (readIfCompat(ON, "CONFLICT", "DO", "NOTHING")) {
    command.setIgnore(true);
}
```

Four tokens. Exactly four. No column list. H2's `ON CONFLICT` implementation is the no-target form only — `ON CONFLICT DO NOTHING` as a four-word phrase, nothing more. PostgreSQL accepts both forms. The `ledger_subject_sequence` table has one unique constraint, so the no-target form works correctly on both databases. Drop the column list, it compiles, tests pass. The dialect split disappears.

**The `ActorIdentityProvider` move was overdue, and I knew it the moment I ran the finder.** The `consumer-spi-placement` protocol has been in the garden for months: any interface an external consumer is expected to implement must live in `api/spi/`, not `runtime/`. `ActorIdentityProvider` was in `runtime/privacy/`. Anyone supplying a custom pseudonymisation strategy would have to depend on the full runtime — JPA entities, CDI beans, build-time deps. Moving it to `api/spi/` took IntelliJ's move-file refactoring about thirty seconds.

The return type change on `tokeniseForQuery` was subtler. The old behaviour — return the raw actorId when no mapping exists — was the right default for SYSTEM and AGENT actors, who are stored under their raw identity. But returning a `String` gave callers no way to distinguish "got a token" from "no mapping, here's the raw value back." The proposed fix was `Optional.empty()` for "no mapping." That breaks SYSTEM actor queries immediately — their entries exist in the ledger under the raw actorId, so returning empty would cause `findByActorId` to skip the query entirely.

The correct semantics: `Optional.empty()` for null input only. Present for everything else, whether that's a UUID token or the raw actorId. Callers get explicit null handling; SYSTEM actors still work; distinguishing "token vs raw" is a comparison, not an API guarantee. A new protocol formalises this for future `ActorIdentityProvider` implementors.

**The TrustScoreCache deletion in casehub-engine was the cleanest change.** The engine had been maintaining its own in-memory cache of capability trust scores — `TrustScoreCache`, an `@Startup @ApplicationScoped` bean that hydrated from the repository at boot and refreshed on CDI events. `CachedTrustScoreSource` in the ledger runtime does the same thing, more completely: it handles both `TrustScoreFullPayload` (batch refresh) and `TrustScoreActorUpdatedEvent` (incremental refresh), which the engine's cache was missing. Delete `TrustScoreCache`, inject `TrustScoreSource`, map `cache.getCapabilityScore()` → `source.capabilityScore()`. The engine tests needed `CachedTrustScoreSource` added to `selected-alternatives`, and one test that called `trustScoreCache.onFull()` needed to inject the concrete type rather than the interface. Six hundred lines gone.

The batch scoring API for `TrustScoreSource` (#136) was the one change that felt most like a straightforward design extension. `scoresFor(List<String> candidateIds, String capabilityTag)` as a `default` method with a loop fallback, overridable for the JPA path with a single `WHERE actorId IN (...)` query. The only real decision was whether the batch method belonged on `TrustScoreSource` or only on `TrustGateService`. It belongs on the source — implementations can optimise the access pattern. `TrustGateService` is a thin delegate.

The H2 source-reading technique is worth keeping. Next time a `MODE=PostgreSQL` compatibility question comes up and the docs are silent, start with `Parser.java`.
