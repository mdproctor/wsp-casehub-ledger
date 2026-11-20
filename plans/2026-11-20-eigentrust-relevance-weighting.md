# EigenTrust Relevance Weighting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #238 — apply relevance weighting to EigenTrust edge weights
**Issue group:** #238

**Goal:** Apply decay, confidence, and relevance weighting to EigenTrust's local-trust matrix edges so that stale, low-confidence, and domain-irrelevant attestations contribute proportionally less to transitive trust scores.

**Architecture:** Add a `Map<UUID, Double> edgeWeights` parameter to `EigenTrustComputer.compute()`. Each caller computes per-attestation effective weights as `decay × confidence × relevance` and passes them through. EigenTrustComputer applies the weight in its matrix-building loop instead of flat ±1.0. Row normalisation already ensures the matrix remains row-stochastic.

**Tech Stack:** Java 21, pure JUnit 5 (ledger-core), Quarkus CDI (runtime)

## Global Constraints

- `ledger-core` is framework-neutral — no CDI, no Quarkus imports
- `EigenTrustComputer` stays algorithm-agnostic: it accepts pre-computed weights, does not know about decay/relevance/confidence as concepts
- Credibility is NOT applied (EigenTrust handles attestor trust transitively via power iteration)
- Default weight is 1.0 for any attestation not in the map (backward compatible)

---

## Batch 1: EigenTrust weighted edge support

### Task 1: EigenTrustComputer accepts and applies edge weights

**Files:**
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/trust/EigenTrustComputer.java:63,91-105`
- Create: `ledger-core/src/test/java/io/casehub/ledger/core/trust/EigenTrustComputerTest.java`

**Interfaces:**
- Produces: `EigenTrustComputer.compute(List<LedgerAttestation>, Map<UUID, String>, Set<String>, Map<UUID, Double>)` → `Map<String, Double>`

- [ ] **Step 1: Write failing tests**

Create `EigenTrustComputerTest.java`. Three test cases:

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class EigenTrustComputerTest {

    private final EigenTrustComputer computer = new EigenTrustComputer(0.5);

    private LedgerAttestation attestation(UUID id, String attestorId,
            UUID entryId, AttestationVerdict verdict, double confidence) {
        var att = new LedgerAttestation();
        att.id = id;
        att.attestorId = attestorId;
        att.ledgerEntryId = entryId;
        att.verdict = verdict;
        att.confidence = confidence;
        att.occurredAt = Instant.now();
        return att;
    }

    @Test
    void uniformWeightsMatchUnweighted() {
        // Two attestors each giving SOUND on the other's entry
        UUID e1 = UUID.randomUUID(), e2 = UUID.randomUUID();
        UUID a1 = UUID.randomUUID(), a2 = UUID.randomUUID();
        var att1 = attestation(a1, "alice", e2, AttestationVerdict.SOUND, 1.0);
        var att2 = attestation(a2, "bob", e1, AttestationVerdict.SOUND, 1.0);

        Map<UUID, String> entryActorIndex = Map.of(e1, "alice", e2, "bob");
        var atts = List.of(att1, att2);

        Map<String, Double> unweighted = computer.compute(atts, entryActorIndex, Set.of());
        Map<String, Double> weighted = computer.compute(atts, entryActorIndex, Set.of(),
                Map.of(a1, 1.0, a2, 1.0));

        assertEquals(unweighted.get("alice"), weighted.get("alice"), 1e-9);
        assertEquals(unweighted.get("bob"), weighted.get("bob"), 1e-9);
    }

    @Test
    void highWeightAttestationDominates() {
        // charlie attests on alice's entry (SOUND, weight 1.0)
        // dave attests on alice's entry (FLAGGED, weight 0.1 — low relevance)
        UUID eAlice = UUID.randomUUID();
        UUID ac = UUID.randomUUID(), ad = UUID.randomUUID();
        var attCharlie = attestation(ac, "charlie", eAlice, AttestationVerdict.SOUND, 1.0);
        var attDave = attestation(ad, "dave", eAlice, AttestationVerdict.FLAGGED, 1.0);

        // Also need entries for charlie and dave so they appear as actors
        UUID eCharlie = UUID.randomUUID(), eDave = UUID.randomUUID();
        Map<UUID, String> entryActorIndex = Map.of(
                eAlice, "alice", eCharlie, "charlie", eDave, "dave");

        var atts = List.of(attCharlie, attDave);
        Map<UUID, Double> weights = Map.of(ac, 1.0, ad, 0.1);

        Map<String, Double> scores = computer.compute(atts, entryActorIndex, Set.of(), weights);

        // alice should have positive trust — charlie's SOUND (weight 1.0) dominates
        // dave's FLAGGED (weight 0.1)
        assertTrue(scores.getOrDefault("alice", 0.0) > 0.0,
                "alice should have positive trust when high-weight SOUND dominates low-weight FLAGGED");
    }

    @Test
    void emptyWeightsMapDefaultsToUnweighted() {
        UUID e1 = UUID.randomUUID(), a1 = UUID.randomUUID();
        var att = attestation(a1, "alice", e1, AttestationVerdict.SOUND, 1.0);

        Map<UUID, String> entryActorIndex = Map.of(e1, "bob");
        var atts = List.of(att);

        Map<String, Double> withEmpty = computer.compute(atts, entryActorIndex, Set.of(), Map.of());
        Map<String, Double> without = computer.compute(atts, entryActorIndex, Set.of());

        assertEquals(without.get("alice"), withEmpty.get("alice"), 1e-9);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=EigenTrustComputerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — 4-arg `compute()` doesn't exist yet

- [ ] **Step 3: Add overloaded compute method with edgeWeights parameter**

In `EigenTrustComputer.java`, add a new 4-arg overload and have the existing 3-arg delegate to it with an empty map:

```java
// Existing method becomes a delegate
public Map<String, Double> compute(List<LedgerAttestation> allAttestations,
                                    Map<UUID, String> entryActorIndex,
                                    Set<String> preTrustedActors) {
    return compute(allAttestations, entryActorIndex, preTrustedActors, Map.of());
}

// New overload with edge weights
public Map<String, Double> compute(List<LedgerAttestation> allAttestations,
                                    Map<UUID, String> entryActorIndex,
                                    Set<String> preTrustedActors,
                                    Map<UUID, Double> edgeWeights) {
    // ... existing body, but lines 101-104 change:
```

In the matrix-building loop (lines 91-106), replace flat ±1.0 with weighted values:

```java
final double weight = edgeWeights.getOrDefault(att.id, 1.0);
if (att.verdict == AttestationVerdict.SOUND || att.verdict == AttestationVerdict.ENDORSED) {
    s[i][j] += weight;
} else if (att.verdict == AttestationVerdict.FLAGGED || att.verdict == AttestationVerdict.CHALLENGED) {
    s[i][j] -= weight;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=EigenTrustComputerTest`
Expected: 3 tests PASS

- [ ] **Step 5: Run full ledger-core test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core`
Expected: all tests pass (existing callers use the 3-arg delegate, no breakage)

- [ ] **Step 6: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/trust/EigenTrustComputer.java \
        ledger-core/src/test/java/io/casehub/ledger/core/trust/EigenTrustComputerTest.java
git commit -m "feat(eigentrust): accept edge weights in compute method

Add 4-arg compute() overload accepting Map<UUID, Double> edgeWeights.
Matrix building loop applies weight instead of flat ±1.0.
Existing 3-arg method delegates with empty map (backward compatible).

Refs #238"
```

---

## Batch 2: Wire weight computation into EigenTrust callers

### Task 2: Compute effective weights and pass to EigenTrust

**Files:**
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputationService.java:32-49,116-136`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java:169-194`
- Create: `ledger-core/src/test/java/io/casehub/ledger/core/trust/EigenTrustEdgeWeightComputationTest.java`

**Interfaces:**
- Consumes: `EigenTrustComputer.compute(List<LedgerAttestation>, Map<UUID, String>, Set<String>, Map<UUID, Double>)`
- Consumes: `DecayFunction.weight(long ageInDays, AttestationVerdict verdict)` → `double`
- Consumes: `RelevanceComposer.assess(List<AttestationRelevanceContext>)` → `Map<UUID, Double>`
- Consumes: `LedgerAttestation.confidence` (double), `LedgerAttestation.occurredAt` (Instant)

- [ ] **Step 1: Write a unit test for edge weight computation logic**

Extract the weight computation into a static utility method so it can be tested independently of CDI wiring. Create a test:

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class EigenTrustEdgeWeightComputationTest {

    @Test
    void computesProductOfThreeFactors() {
        Instant now = Instant.now();
        UUID attId = UUID.randomUUID();

        var att = new LedgerAttestation();
        att.id = attId;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 0.8;
        att.occurredAt = now.minus(10, ChronoUnit.DAYS);

        double decayWeight = 0.5;
        double relevanceWeight = 0.7;

        // decay × confidence × relevance = 0.5 × 0.8 × 0.7 = 0.28
        DecayFunction decay = (age, verdict) -> decayWeight;
        Map<UUID, Double> relevance = Map.of(attId, relevanceWeight);

        Map<UUID, Double> weights = EigenTrustEdgeWeights.compute(
                List.of(att), decay, relevance, now);

        assertEquals(0.28, weights.get(attId), 1e-9);
    }

    @Test
    void missingRelevanceDefaultsToOne() {
        Instant now = Instant.now();
        UUID attId = UUID.randomUUID();

        var att = new LedgerAttestation();
        att.id = attId;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 0.9;
        att.occurredAt = now;

        DecayFunction decay = (age, verdict) -> 1.0;

        Map<UUID, Double> weights = EigenTrustEdgeWeights.compute(
                List.of(att), decay, Map.of(), now);

        // decay(1.0) × confidence(0.9) × relevance(1.0) = 0.9
        assertEquals(0.9, weights.get(attId), 1e-9);
    }

    @Test
    void zeroConfidenceProducesZeroWeight() {
        Instant now = Instant.now();
        UUID attId = UUID.randomUUID();

        var att = new LedgerAttestation();
        att.id = attId;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 0.0;
        att.occurredAt = now;

        DecayFunction decay = (age, verdict) -> 1.0;

        Map<UUID, Double> weights = EigenTrustEdgeWeights.compute(
                List.of(att), decay, Map.of(), now);

        assertEquals(0.0, weights.get(attId), 1e-9);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=EigenTrustEdgeWeightComputationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `EigenTrustEdgeWeights` doesn't exist

- [ ] **Step 3: Implement EigenTrustEdgeWeights utility**

Create `ledger-core/src/main/java/io/casehub/ledger/core/trust/EigenTrustEdgeWeights.java`:

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.model.LedgerAttestation;

import java.time.Duration;
import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public final class EigenTrustEdgeWeights {

    private EigenTrustEdgeWeights() {}

    public static Map<UUID, Double> compute(List<LedgerAttestation> attestations,
                                             DecayFunction decayFunction,
                                             Map<UUID, Double> relevanceByAttestationId,
                                             Instant now) {
        Map<UUID, Double> weights = new HashMap<>();
        for (LedgerAttestation att : attestations) {
            long ageInDays = att.occurredAt != null
                    ? Duration.between(att.occurredAt, now).toDays()
                    : 0;
            double decay = decayFunction.weight(ageInDays, att.verdict);
            double confidence = att.confidence;
            double relevance = relevanceByAttestationId.getOrDefault(att.id, 1.0);
            weights.put(att.id, decay * confidence * relevance);
        }
        return weights;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=EigenTrustEdgeWeightComputationTest`
Expected: 3 tests PASS

- [ ] **Step 5: Wire TrustScoreComputationService to pass weights**

In `TrustScoreComputationService.java`:

1. Add `DecayFunction` and `RelevanceComposer` constructor parameters (fields + constructor args)
2. In `runEigenTrustPass()`, after building `allAttestations` (line 122-124):

```java
Map<UUID, Double> relevanceMap = relevanceComposer.assess(
        allAttestations.stream()
                .map(att -> new AttestationRelevanceContext(
                        att.id, att.attestorId, att.attestorType, att.attestorRole,
                        entryActorIndex.get(att.ledgerEntryId), null, att.capabilityTag,
                        att.trustDimension, att.occurredAt, Map.of()))
                .toList());

Map<UUID, Double> edgeWeights = EigenTrustEdgeWeights.compute(
        allAttestations, decayFunction, relevanceMap, Instant.now());

Map<String, Double> globalScores = eigenTrustComputer.compute(
        allAttestations, entryActorIndex, preTrustedActors, edgeWeights);
```

- [ ] **Step 6: Wire TrustScoreJob to pass weights**

In `TrustScoreJob.java`:

1. Add `@Inject DecayFunction decayFunction` and `@Inject RelevanceComposer relevanceComposer` fields
2. In `runEigenTrustPass()`, after building `allAttestations` (line 177-179), add the same weight computation as Step 5
3. Pass `edgeWeights` to `eigenTrust.compute()` (line 188)

- [ ] **Step 7: Install ledger-core and run runtime tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -DskipTests -q
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all 868+ tests pass

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core`
Expected: all tests pass including new EigenTrust tests

- [ ] **Step 9: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/trust/EigenTrustEdgeWeights.java \
        ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputationService.java \
        ledger-core/src/test/java/io/casehub/ledger/core/trust/EigenTrustEdgeWeightComputationTest.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
git commit -m "feat(eigentrust): apply decay × confidence × relevance edge weights

Compute per-attestation effective weight (decay × confidence × relevance)
via EigenTrustEdgeWeights utility. Both TrustScoreComputationService and
TrustScoreJob pass weights through to EigenTrustComputer.

Credibility excluded — EigenTrust handles attestor trust transitively
via power iteration.

Refs #238"
```

---

## References

- [2026-11-20-eigentrust-relevance-weighting-design.md] — design spec this plan implements
- `ledger-core/.../trust/EigenTrustComputer.java:63,91-106` — matrix building loop (injection point)
- `ledger-core/.../trust/TrustScoreComputationService.java:116-136` — ledger-core EigenTrust caller
- `runtime/.../service/TrustScoreJob.java:169-194` — runtime EigenTrust caller
- `ledger-core/.../trust/DecayFunction.java:6-9` — decay weight SPI
- `ledger-core/.../trust/RelevanceComposer.java:33` — relevance batch assessment
- `ledger-core/.../trust/AttestationWeights.java` — existing weight container (relevanceByAttestationId)
- `api/.../model/LedgerAttestation.java:34,47` — confidence and occurredAt fields
- GitHub #238 — focal issue
- GitHub #231 — parent issue (attestation relevance SPI)
