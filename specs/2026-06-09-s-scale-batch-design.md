# S-Scale Batch: #124, #125, #129

Three S-scale issues implemented on a single branch.

---

## #124 — Combined attestation query for ComputedTrustScoreSource

### Problem

`ComputedTrustScoreSource.computeFresh()` and `IncrementalTrustUpdateObserver.onAttestationRecorded()` both use a two-step pattern to load an actor's attestation data:

1. `ledgerRepo.findEventsByActorId(actorId)` → `List<LedgerEntry>`
2. Extract entry IDs → `ledgerRepo.findAttestationsForEntries(entryIds)` → `Map<UUID, List<LedgerAttestation>>`

This is two round-trips and requires the caller to extract and pass entry IDs manually.

### Design

Add `findAttestationsByActorId(String actorId)` to `CrossTenantLedgerEntryRepository`. Returns `Map<UUID, List<LedgerAttestation>>` — attestations grouped by entry ID, filtered to entries where `entryType = EVENT` and `actorId` matches.

**JPA implementation** (`JpaCrossTenantLedgerEntryRepository`): single JPQL query joining `LedgerAttestation a` to `LedgerEntry e` on `a.ledgerEntryId = e.id` where `e.actorId = :actorId AND e.entryType = EVENT`. Groups results in Java (Collectors.groupingBy).

**In-memory implementation** (`InMemoryCrossTenantLedgerEntryRepository`): filter `allEntries()` for actorId + EVENT, collect IDs, then filter `allAttestations()` by those IDs.

**In-memory reactive** (`InMemoryCrossTenantReactiveLedgerEntryRepository`): delegate to blocking.

**Callers updated:**
- `ComputedTrustScoreSource.computeFresh()` — replace two-step with single call. Still needs `findEventsByActorId()` for the decisions list passed to `calculator.computeAll()`.
- `IncrementalTrustUpdateObserver.onAttestationRecorded()` — same replacement.

### Per-capability query: deferred

The original issue proposed per-capability filtering at the DB level. This is redundant given the per-actor cache in `ComputedTrustScoreSource` — after the first call, all subsequent score queries for the same actor are served from `ComputedScores` without hitting the DB. The combined query addresses the real inefficiency (two round-trips). Per-capability filtering adds complexity with no measurable benefit at current scale.

---

## #125 — Materialization flag for computed-only deployments

### Problem

When `ComputedTrustScoreSource` is active, `TrustScoreJob` and `IncrementalTrustUpdateObserver` still run — computing and writing scores to `ActorTrustScoreRepository` that nobody reads. This is harmless but wasteful for lightweight deployments.

The existing `casehub.ledger.trust-score.enabled` flag gates the entire trust feature, including config defaults. A deployment using `ComputedTrustScoreSource` may still want `trust-score.enabled=true` for configuration (decay, aggregation strategy) while skipping materialization.

### Design

Add `materialization` sub-config to `LedgerConfig.TrustScoreConfig`:

```java
interface MaterializationConfig {
    @WithDefault("true")
    boolean enabled();
}
```

Config key: `casehub.ledger.trust-score.materialization.enabled` (default `true`).

**When `false`:**
- `TrustScoreJob.computeTrustScores()` returns immediately after the `enabled` check — skips `runComputation()` entirely (Beta, bootstrap, EigenTrust, routing publisher).
- `IncrementalTrustUpdateObserver.onAttestationRecorded()` returns immediately — adds `materialization.enabled` to the existing gate check.

**When `true` (default):** no behaviour change.

`TrustScoreRoutingPublisher` is not gated — it only fires when `TrustScoreJob` calls `publish()`. With materialization disabled, the job never reaches `publish()`.

### Deployment matrix

| trust-score.enabled | materialization.enabled | TrustScoreSource | Effect |
|---------------------|------------------------|------------------|--------|
| false | (irrelevant) | any | No computation at all — job/observer skip |
| true | true (default) | Materialized/Cached | Full write path — current behaviour |
| true | false | Computed | Config active, no writes — computed reads only |
| true | true | Computed | Both paths run — exports/routing from materialized, reads from computed |

---

## #129 — Carry tenancyId through CDI event chain

### Problem

`ActorIdentityBindingObserver` persists `ActorIdentityBindingEntry` via `@ObservesAsync` without tenant context. The `@ObservesAsync` handler has no CDI request scope, so there is no way to resolve tenancyId from `CurrentPrincipal`. The interim fix defaults to `TenancyConstants.DEFAULT_TENANT_ID` in `JpaActorIdentityBindingRepository.save()`.

### Design

**1. casehub-platform-api — event records gain tenancyId**

Per governing protocol PP-20260601-e368ea (tenancyId as 2nd component):

```java
// Before
public record AgentIdentityValidatedEvent(
    String actorId, String actorDid, IdentityBindingStatus status,
    boolean alsoKnownAsVerified, boolean keyMatchVerified,
    String verifiedKeyRef, CredentialValidationResult credentialResult,
    String didMethod) {}

// After
public record AgentIdentityValidatedEvent(
    String actorId, String tenancyId, String actorDid, IdentityBindingStatus status,
    boolean alsoKnownAsVerified, boolean keyMatchVerified,
    String verifiedKeyRef, CredentialValidationResult credentialResult,
    String didMethod) {}
```

Same for `AgentIdentityViolationEvent`:

```java
// Before
public record AgentIdentityViolationEvent(
    String actorId, String actorDid, IdentityBindingStatus status) {}

// After
public record AgentIdentityViolationEvent(
    String actorId, String tenancyId, String actorDid,
    IdentityBindingStatus status) {}
```

**2. ledger runtime — ActorIdentityValidationEnricher.fireEvent()**

Pass `entry.tenancyId` when constructing events:

```java
event.fireAsync(new AgentIdentityValidatedEvent(
    entry.actorId, entry.tenancyId, entry.actorDid, status,
    true, true, entry.agentKeyRef, null, didMethod));

event.fireAsync(new AgentIdentityViolationEvent(
    entry.actorId, entry.tenancyId, entry.actorDid, status));
```

**3. ledger runtime — ActorIdentityBindingObserver**

Read `tenancyId` from event and set on entry:

```java
void onValidated(@ObservesAsync AgentIdentityValidatedEvent event) {
    persistBinding(event.tenancyId(), event.actorId(), event.actorDid(), ...);
}
```

`persistBinding()` gains a `tenancyId` parameter and sets `entry.tenancyId = tenancyId`.

**4. JpaActorIdentityBindingRepository.save() — remove fallback**

Remove the `if (entry.tenancyId == null)` fallback. The tenancyId is now always provided through the event chain. A null tenancyId at save time is a bug — let it fail visibly rather than silently defaulting.

### Cross-repo sequence

1. Edit `AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent` in `casehub-platform` repo.
2. `mvn install` casehub-platform-api to publish SNAPSHOT to local `.m2`.
3. Consume in ledger — update enricher, observer, repository.
4. File downstream issues for any other consumers of these events (search cross-repo).

### Downstream impact

Search all repos for consumers of `AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent`. Known consumer: `ActorIdentityBindingObserver` (this repo). Any others will need to add the `tenancyId` parameter to their constructor calls — the record component addition is a compile break by design.
