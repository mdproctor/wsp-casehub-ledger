---
layout: post
title: "The Tenant Whose Key Was Always the Same"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [multi-tenancy, merkle, cdi, quarkus, h2]
---

Three fixes landed on this branch. One of them turned out to be about cryptographic correctness, not health monitoring.

`KeyRotationEntry` and `ActorIdentityBindingEntry` both derive their `subjectId` from `UUID.nameUUIDFromBytes(actorId)`. Two tenants sharing the same `actorId` — `claude:reviewer@v1` running across two organisations, for instance — produce the same UUID. The multi-tenancy design documented this case and noted that the Merkle frontier table and sequence counter table were "scoped by subjectId which is already unique per tenant." That assumption held for domain aggregates. It doesn't hold for identity-derived subjects.

The obvious consequence is frontier row collision: both tenants overwrite each other's Merkle state on every save. The less obvious consequence is a proof problem. `LedgerVerificationService.inclusionProof()` computes the Merkle leaf position as `k = entry.sequenceNumber - 1`. With a shared counter, tenant A gets sequence numbers 1, 3, 5 — making `k = 0, 2, 4`. But tenant A's Merkle tree has three leaves, at positions 0, 1, 2. The proof is constructed for the wrong position. Verification fails with a wrong root hash rather than a useful error.

Adding `tenancy_id` to both tables as part of their primary keys fixes it. The sequence counter advances independently per tenant, frontier rows are keyed on `(subject_id, tenancy_id)`, and inclusion proofs get the right leaf index.

Threading this through the H2 MERGE statement produced an unexpected gotcha. The updated USING subquery looked like:

```sql
USING (SELECT CAST(?1 AS UUID) AS sid, ?2 AS tid) AS s
```

H2 threw `Unknown data type: "TID"`. It was interpreting `tid` as a PostgreSQL type name — TID is the PostgreSQL tuple ID type — rather than a column alias. `CAST(?2 AS VARCHAR) AS tval` resolved it. H2 in `MODE=PostgreSQL` parses alias names in USING subqueries as potential type casts. This is not documented anywhere.

The health job gap detection also needed updating. The JPQL grouped by `subjectId` alone, which would produce false positives for shared nameUUID subjects after this fix. The natural extension would have been adding nullable `tenancyId` to `LedgerGapDetected`, but the record already conflated two incompatible shapes: `subjectId` stored a UUID for sequence gaps and a type-name string for reconciliation mismatches. A sealed `LedgerAnomalyDetected` interface with two concrete records — `LedgerSequenceGapDetected` and `LedgerReconciliationMismatchDetected` — is the right design. Observers pattern-match on the concrete type instead of checking a discriminator enum.

The other two issues were lighter. `JpaActorTrustScoreRepository` was plain `@ApplicationScoped`, which forced consumers without a datasource to use `quarkus.arc.exclude-types` to prevent startup failures. Adding `@Alternative` alongside a new `NoOpActorTrustScoreRepository @DefaultBean` applies the same three-tier CDI pattern from #138. The dialect detection fix addressed a named-datasource scenario: `@ConfigProperty(name = "quarkus.datasource.db-kind")` reads the default datasource key only. A consumer setting `casehub.ledger.datasource=myds` silently gets `"h2"` returned even against a PostgreSQL connection. `em.unwrap(Session.class).doReturningWork(conn -> conn.getMetaData().getDatabaseProductName()...)` removes the configuration coupling entirely.

The nameUUID assumption was documented in the original design. It was marked correct because domain aggregate UUIDs are globally unique. The assumption just didn't hold for two specific entry types where identity-derived UUIDs are the design. A new protocol makes this explicit so future per-subject tables don't inherit the same gap.
