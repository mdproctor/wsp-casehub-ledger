# Trust Score Routing Signals — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `quarkus.ledger.trust-score.routing-enabled` — fire CDI events after each trust score computation so routing consumers (e.g. CaseHub task assignment) can observe trust scores using whichever granularity they need.

**Architecture:** Three CDI event payload types (`TrustScoreFullPayload`, `TrustScoreDeltaPayload`, `TrustScoreComputedAt`) act as strategy selectors — consumers observe whichever type fits. A new `TrustScoreRoutingPublisher` CDI bean handles observer detection at startup and dispatch after each `TrustScoreJob` run. Sync vs async dispatch is per-consumer using standard `@Observes` / `@ObservesAsync`.

**Tech Stack:** Java 21 records, Quarkus CDI (`jakarta.enterprise.event.Event`, `BeanManager`), JUnit 5, AssertJ, `@QuarkusTest` with isolated H2 profiles.

---

## File Map

**New — runtime module:**
```
runtime/src/main/java/io/casehub/ledger/runtime/service/routing/
├── TrustScoreDelta.java              — record: actorId + previous/new score pair
├── TrustScoreComputedAt.java         — record: Instant computedAt + int actorCount
├── TrustScoreFullPayload.java        — record: immutable List<ActorTrustScore>
├── TrustScoreDeltaPayload.java       — record: List<TrustScoreDelta>
└── TrustScoreRoutingPublisher.java   — @ApplicationScoped, holds Event<T> per type, dispatches

runtime/src/test/java/io/casehub/ledger/service/routing/
├── TrustScoreRoutingPublisherTest.java  — pure JUnit 5, tests computeDeltas static method
├── TestRoutingObservers.java            — @ApplicationScoped test CDI bean, captures received events
├── TrustScoreRoutingIT.java             — @QuarkusTest, routing-enabled=true
└── TrustScoreRoutingDisabledIT.java     — @QuarkusTest, routing-enabled=false
```

**Modified — runtime module:**
```
runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
  — add routingDeltaThreshold() to TrustScoreConfig

runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
  — inject TrustScoreRoutingPublisher, pre-read snapshot, call publish() at end of runComputation()

runtime/src/test/resources/application.properties
  — add %routing-test and %routing-disabled-test profiles
```

**New — example module:**
```
examples/trust-score-routing/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/io/casehub/ledger/examples/routing/
    │   │   ├── ledger/TaskLedgerEntry.java
    │   │   ├── routing/TaskRouter.java           — @Observes TrustScoreFullPayload → ranked list
    │   │   ├── routing/RoutingSignalLogger.java  — @ObservesAsync TrustScoreComputedAt → log
    │   │   └── api/TaskRoutingResource.java      — GET /routing/ranked-agents
    │   └── resources/
    │       ├── application.properties
    │       └── db/migration/V2000__task_ledger_entry.sql
    └── test/java/io/casehub/ledger/examples/routing/
        └── TrustScoreRoutingE2EIT.java
```

**Modified — root:**
```
pom.xml                — add examples/trust-score-routing module
docs/DESIGN.md         — fix staleness, update tracker
docs/RESEARCH.md       — mark items #10 and #11 done, update What's Next
```

---

## Task 0: Create GitHub epic and issues

- [ ] **Create the epic**

```bash
gh issue create \
  --title "Trust score routing signals" \
  --body "Implement quarkus.ledger.trust-score.routing-enabled. Fire CDI events (TrustScoreFullPayload, TrustScoreDeltaPayload, TrustScoreComputedAt) after each TrustScoreJob run. Consumers select strategy by observing the payload type they need. Sync/async per-consumer via @Observes / @ObservesAsync." \
  --label "enhancement"
```

Record the epic issue number as `EPIC` (e.g. `#32`).

- [ ] **Create child issues**

```bash
gh issue create \
  --title "Implement TrustScoreRoutingPublisher and payload types" \
  --body "Refs EPIC. Payload records (TrustScoreDelta, TrustScoreComputedAt, TrustScoreFullPayload, TrustScoreDeltaPayload), TrustScoreRoutingPublisher CDI bean, LedgerConfig.routingDeltaThreshold(), TrustScoreJob wiring. TDD: unit tests for computeDeltas, IT for CDI dispatch." \
  --label "enhancement"

gh issue create \
  --title "E2E example module: trust-score-routing" \
  --body "Refs EPIC. New examples/trust-score-routing demonstrating all three payload types with a simulated task assignment scenario. E2E IT exercises the full pipeline." \
  --label "enhancement"

gh issue create \
  --title "Fix DESIGN.md and RESEARCH.md staleness" \
  --body "OTel correlation wiring tracker still shows Pending (closed #31). RESEARCH.md items #10 and #11 still show open (both done). Update tracker with routing signals entry. Update last-updated dates." \
  --label "documentation"
```

Record all three issue numbers. Use them in every commit below.

---

## Task 1: Payload value type records

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreDelta.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreComputedAt.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreFullPayload.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreDeltaPayload.java`

- [ ] **Write TrustScoreDelta**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreDelta.java
package io.casehub.ledger.runtime.service.routing;

public record TrustScoreDelta(
        String actorId,
        double previousScore,
        double newScore,
        double previousGlobalScore,
        double newGlobalScore) {
}
```

- [ ] **Write TrustScoreComputedAt**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreComputedAt.java
package io.casehub.ledger.runtime.service.routing;

import java.time.Instant;

public record TrustScoreComputedAt(Instant computedAt, int actorCount) {
}
```

- [ ] **Write TrustScoreFullPayload**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreFullPayload.java
package io.casehub.ledger.runtime.service.routing;

import java.util.List;

import io.casehub.ledger.runtime.model.ActorTrustScore;

public record TrustScoreFullPayload(List<ActorTrustScore> scores) {
}
```

- [ ] **Write TrustScoreDeltaPayload**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreDeltaPayload.java
package io.casehub.ledger.runtime.service.routing;

import java.util.List;

public record TrustScoreDeltaPayload(List<TrustScoreDelta> deltas) {
}
```

- [ ] **Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/routing/
git commit -m "feat: add routing signal payload types (TrustScoreDelta, TrustScoreComputedAt, TrustScoreFullPayload, TrustScoreDeltaPayload)

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 2: Unit tests for computeDeltas (write failing first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingPublisherTest.java`

The `computeDeltas` method does not exist yet — these tests will fail to compile until Task 3.

- [ ] **Write the unit tests**

```java
// runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingPublisherTest.java
package io.casehub.ledger.service.routing;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.service.routing.TrustScoreDelta;
import io.casehub.ledger.runtime.service.routing.TrustScoreRoutingPublisher;

class TrustScoreRoutingPublisherTest {

    // ── Fixture ───────────────────────────────────────────────────────────────

    private static ActorTrustScore score(final String actorId,
            final double trustScore, final double globalScore) {
        final ActorTrustScore s = new ActorTrustScore();
        s.actorId = actorId;
        s.trustScore = trustScore;
        s.globalTrustScore = globalScore;
        return s;
    }

    // ── Happy path ────────────────────────────────────────────────────────────

    @Test
    void computeDeltas_firstRun_allActorsIncludedWithZeroPrevious() {
        final var current = List.of(
                score("agent-a", 0.7, 0.4),
                score("agent-b", 0.3, 0.1));
        final var previous = Map.<String, ActorTrustScore> of();

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).hasSize(2);
        assertThat(deltas).allSatisfy(d -> {
            assertThat(d.previousScore()).isEqualTo(0.0);
            assertThat(d.previousGlobalScore()).isEqualTo(0.0);
        });
    }

    @Test
    void computeDeltas_onlyChangedActorsIncluded() {
        final var current = List.of(
                score("agent-a", 0.8, 0.0),
                score("agent-b", 0.5, 0.0));
        final var previous = Map.of(
                "agent-a", score("agent-a", 0.5, 0.0),
                "agent-b", score("agent-b", 0.5, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).hasSize(1);
        assertThat(deltas.get(0).actorId()).isEqualTo("agent-a");
    }

    // ── Correctness ───────────────────────────────────────────────────────────

    @Test
    void computeDeltas_changeBelowThreshold_actorExcluded() {
        final var current = List.of(score("agent-a", 0.51, 0.0));
        final var previous = Map.of("agent-a", score("agent-a", 0.50, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.02);

        assertThat(deltas).isEmpty();
    }

    @Test
    void computeDeltas_changeAtThreshold_actorIncluded() {
        final var current = List.of(score("agent-a", 0.52, 0.0));
        final var previous = Map.of("agent-a", score("agent-a", 0.50, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.02);

        assertThat(deltas).hasSize(1);
        assertThat(deltas.get(0).previousScore()).isCloseTo(0.50, within(0.001));
        assertThat(deltas.get(0).newScore()).isCloseTo(0.52, within(0.001));
    }

    @Test
    void computeDeltas_noChange_emptyDeltas() {
        final var current = List.of(score("agent-a", 0.5, 0.0));
        final var previous = Map.of("agent-a", score("agent-a", 0.5, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).isEmpty();
    }

    @Test
    void computeDeltas_capturesPreviousAndNewGlobalScore() {
        final var current = List.of(score("agent-a", 0.8, 0.6));
        final var previous = Map.of("agent-a", score("agent-a", 0.5, 0.3));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).hasSize(1);
        final TrustScoreDelta d = deltas.get(0);
        assertThat(d.previousGlobalScore()).isCloseTo(0.3, within(0.001));
        assertThat(d.newGlobalScore()).isCloseTo(0.6, within(0.001));
    }

    // ── Robustness ────────────────────────────────────────────────────────────

    @Test
    void computeDeltas_emptyCurrentScores_returnsEmpty() {
        final var deltas = TrustScoreRoutingPublisher.computeDeltas(
                List.of(), Map.of(), 0.01);

        assertThat(deltas).isEmpty();
    }

    @Test
    void computeDeltas_negativeScoreChange_belowThreshold_excluded() {
        // Score dropped by 0.005 — below threshold of 0.01 → excluded
        final var current = List.of(score("agent-a", 0.495, 0.0));
        final var previous = Map.of("agent-a", score("agent-a", 0.5, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).isEmpty();
    }

    @Test
    void computeDeltas_negativeScoreChange_atThreshold_included() {
        // Score dropped by 0.01 — at threshold → included
        final var current = List.of(score("agent-a", 0.49, 0.0));
        final var previous = Map.of("agent-a", score("agent-a", 0.5, 0.0));

        final var deltas = TrustScoreRoutingPublisher.computeDeltas(current, previous, 0.01);

        assertThat(deltas).hasSize(1);
        assertThat(deltas.get(0).newScore()).isCloseTo(0.49, within(0.001));
    }
}
```

- [ ] **Run tests to verify they fail to compile (TrustScoreRoutingPublisher doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | tail -5
```

Expected: compilation error mentioning `TrustScoreRoutingPublisher`

---

## Task 3: Implement TrustScoreRoutingPublisher (make unit tests pass)

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreRoutingPublisher.java`

- [ ] **Write TrustScoreRoutingPublisher**

```java
// runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreRoutingPublisher.java
package io.casehub.ledger.runtime.service.routing;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.inject.spi.BeanManager;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.ActorTrustScore;

@ApplicationScoped
public class TrustScoreRoutingPublisher {

    private static final Logger log = Logger.getLogger(TrustScoreRoutingPublisher.class);

    @Inject
    Event<TrustScoreFullPayload> fullEvent;

    @Inject
    Event<TrustScoreDeltaPayload> deltaEvent;

    @Inject
    Event<TrustScoreComputedAt> notifyEvent;

    @Inject
    LedgerConfig config;

    @Inject
    BeanManager beanManager;

    private boolean hasFullObservers;
    private boolean hasDeltaObservers;
    private boolean hasNotifyObservers;

    @PostConstruct
    void detectObservers() {
        hasFullObservers = !beanManager
                .resolveObserverMethods(new TrustScoreFullPayload(List.of())).isEmpty();
        hasDeltaObservers = !beanManager
                .resolveObserverMethods(new TrustScoreDeltaPayload(List.of())).isEmpty();
        hasNotifyObservers = !beanManager
                .resolveObserverMethods(new TrustScoreComputedAt(Instant.EPOCH, 0)).isEmpty();
    }

    /** True when at least one TrustScoreDeltaPayload observer is registered. */
    public boolean needsPreviousSnapshot() {
        return hasDeltaObservers;
    }

    /**
     * Dispatches routing signals to registered observers. Called by TrustScoreJob
     * after each computation run, within the same transaction.
     *
     * @param current         all ActorTrustScore rows after computation
     * @param previousSnapshot scores read before computation (empty map if no delta observers)
     * @param computedAt      timestamp of this computation run
     */
    public void publish(final List<ActorTrustScore> current,
            final Map<String, ActorTrustScore> previousSnapshot,
            final Instant computedAt) {

        if (!config.trustScore().routingEnabled()) {
            return;
        }

        if (hasNotifyObservers) {
            try {
                notifyEvent.fire(new TrustScoreComputedAt(computedAt, current.size()));
            } catch (final Exception e) {
                log.warnf(e, "TrustScoreComputedAt observer failed — routing signal skipped");
            }
        }

        if (hasFullObservers) {
            try {
                fullEvent.fire(new TrustScoreFullPayload(List.copyOf(current)));
            } catch (final Exception e) {
                log.warnf(e, "TrustScoreFullPayload observer failed — routing signal skipped");
            }
        }

        if (hasDeltaObservers) {
            try {
                final double threshold = config.trustScore().routingDeltaThreshold();
                final List<TrustScoreDelta> deltas = computeDeltas(current, previousSnapshot, threshold);
                deltaEvent.fire(new TrustScoreDeltaPayload(deltas));
            } catch (final Exception e) {
                log.warnf(e, "TrustScoreDeltaPayload observer failed — routing signal skipped");
            }
        }
    }

    /**
     * Computes which actors have changed beyond threshold since the previous snapshot.
     * Package-private for unit testing.
     */
    static List<TrustScoreDelta> computeDeltas(
            final List<ActorTrustScore> current,
            final Map<String, ActorTrustScore> previousSnapshot,
            final double threshold) {

        final List<TrustScoreDelta> deltas = new ArrayList<>();
        for (final ActorTrustScore score : current) {
            final ActorTrustScore prev = previousSnapshot.get(score.actorId);
            final double prevTrust = prev != null ? prev.trustScore : 0.0;
            final double prevGlobal = prev != null ? prev.globalTrustScore : 0.0;
            if (Math.abs(score.trustScore - prevTrust) >= threshold) {
                deltas.add(new TrustScoreDelta(
                        score.actorId, prevTrust, score.trustScore,
                        prevGlobal, score.globalTrustScore));
            }
        }
        return deltas;
    }
}
```

- [ ] **Run unit tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreRoutingPublisherTest -q
```

Expected: `Tests run: 9, Failures: 0, Errors: 0`

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreRoutingPublisher.java \
        runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingPublisherTest.java
git commit -m "feat: implement TrustScoreRoutingPublisher with computeDeltas logic

Unit tests: 9 cases covering threshold boundary, first run, global score capture,
negative delta, empty inputs.

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 4: Add routingDeltaThreshold to LedgerConfig

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Add routingDeltaThreshold() to TrustScoreConfig**

In `LedgerConfig.java`, inside `interface TrustScoreConfig`, after `routingEnabled()`:

```java
        /**
         * Minimum absolute change in trust score for an actor to appear in a
         * {@code TrustScoreDeltaPayload}. Prevents noise from floating-point drift.
         *
         * @return delta threshold (default 0.01)
         */
        @WithDefault("0.01")
        double routingDeltaThreshold();
```

- [ ] **Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git commit -m "feat: add quarkus.ledger.trust-score.routing-delta-threshold config key (default 0.01)

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 5: Wire TrustScoreJob to TrustScoreRoutingPublisher

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Add publisher injection and wiring to runComputation()**

Replace the full `TrustScoreJob.java` with:

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.routing.TrustScoreRoutingPublisher;
import io.quarkus.scheduler.Scheduled;

@ApplicationScoped
public class TrustScoreJob {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    LedgerConfig config;

    @Inject
    TrustScoreRoutingPublisher routingPublisher;

    @Scheduled(every = "{quarkus.ledger.trust-score.schedule:24h}", identity = "ledger-trust-score-job")
    @Transactional
    public void computeTrustScores() {
        if (!config.trustScore().enabled()) {
            return;
        }
        runComputation();
    }

    @Transactional
    public void runComputation() {
        final Instant now = Instant.now();

        // Pre-read previous scores only if a delta observer is registered
        final Map<String, ActorTrustScore> previousSnapshot = routingPublisher.needsPreviousSnapshot()
                ? trustRepo.findAll().stream().collect(Collectors.toMap(s -> s.actorId, s -> s))
                : Map.of();

        final TrustScoreComputer computer = new TrustScoreComputer(
                config.trustScore().decayHalfLifeDays());

        final List<LedgerEntry> allEvents = ledgerRepo.findAllEvents();
        final Map<String, List<LedgerEntry>> byActor = allEvents.stream()
                .filter(e -> e.actorId != null)
                .collect(Collectors.groupingBy(e -> e.actorId));

        final Set<UUID> entryIds = allEvents.stream()
                .map(e -> e.id)
                .collect(Collectors.toSet());
        final Map<UUID, List<LedgerAttestation>> attestationsByEntry =
                ledgerRepo.findAttestationsForEntries(entryIds);

        for (final Map.Entry<String, List<LedgerEntry>> actorEntry : byActor.entrySet()) {
            final String actorId = actorEntry.getKey();
            final List<LedgerEntry> decisions = actorEntry.getValue();
            final ActorType actorType = decisions.stream()
                    .map(e -> e.actorType)
                    .filter(t -> t != null)
                    .findFirst()
                    .orElse(ActorType.HUMAN);

            final TrustScoreComputer.ActorScore score =
                    computer.compute(decisions, attestationsByEntry, now);

            trustRepo.upsert(actorId, actorType, score.trustScore(),
                    score.decisionCount(), score.overturnedCount(),
                    score.alpha(), score.beta(),
                    score.attestationPositive(), score.attestationNegative(), now);
        }

        if (config.trustScore().eigentrustEnabled()) {
            runEigenTrustPass(allEvents, attestationsByEntry);
        }

        // Routing signals — after all writes, within the same transaction
        final List<ActorTrustScore> currentScores = trustRepo.findAll();
        routingPublisher.publish(currentScores, previousSnapshot, now);
    }

    private void runEigenTrustPass(
            final List<LedgerEntry> allEvents,
            final Map<UUID, List<LedgerAttestation>> attestationsByEntry) {

        final Map<UUID, String> entryActorIndex = allEvents.stream()
                .filter(e -> e.actorId != null)
                .collect(Collectors.toMap(e -> e.id, e -> e.actorId));

        final List<LedgerAttestation> allAttestations = attestationsByEntry.values().stream()
                .flatMap(List::stream)
                .collect(Collectors.toList());

        final Set<String> preTrustedActors = config.trustScore().preTrustedActors()
                .map(LinkedHashSet::new)
                .orElseGet(LinkedHashSet::new);

        final EigenTrustComputer eigenTrust = new EigenTrustComputer(
                config.trustScore().eigentrustAlpha());

        final Map<String, Double> globalScores = eigenTrust.compute(
                allAttestations, entryActorIndex, preTrustedActors);

        for (final Map.Entry<String, Double> entry : globalScores.entrySet()) {
            trustRepo.updateGlobalTrustScore(entry.getKey(), entry.getValue());
        }
    }
}
```

- [ ] **Run existing TrustScoreIT to verify nothing regressed**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreIT -q
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
git commit -m "feat: wire TrustScoreJob to TrustScoreRoutingPublisher

Pre-reads previous snapshot only when delta observers registered.
Calls publish() after all DB writes within the same transaction.

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 6: Add routing test profiles to application.properties

**Files:**
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Add two new profiles at the end of application.properties**

```properties
# Routing signals test profile — routing enabled (used by TrustScoreRoutingIT)
%routing-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:routingtestdb;DB_CLOSE_DELAY=-1
%routing-test.quarkus.ledger.trust-score.enabled=true
%routing-test.quarkus.ledger.trust-score.routing-enabled=true

# Routing signals disabled test profile (used by TrustScoreRoutingDisabledIT)
%routing-disabled-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:routingdisabledtestdb;DB_CLOSE_DELAY=-1
%routing-disabled-test.quarkus.ledger.trust-score.enabled=true
%routing-disabled-test.quarkus.ledger.trust-score.routing-enabled=false
```

- [ ] **Commit**

```bash
git add runtime/src/test/resources/application.properties
git commit -m "test: add routing-test and routing-disabled-test profiles for TrustScoreRoutingIT

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 7: Write TestRoutingObservers helper bean

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/routing/TestRoutingObservers.java`

This `@ApplicationScoped` bean lives in the test classpath. Quarkus auto-discovers it during `@QuarkusTest` runs. It captures received events and provides a `reset()` method for use between tests.

- [ ] **Write TestRoutingObservers**

```java
// runtime/src/test/java/io/casehub/ledger/service/routing/TestRoutingObservers.java
package io.casehub.ledger.service.routing;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.ObservesAsync;

import io.casehub.ledger.runtime.service.routing.TrustScoreComputedAt;
import io.casehub.ledger.runtime.service.routing.TrustScoreDeltaPayload;
import io.casehub.ledger.runtime.service.routing.TrustScoreFullPayload;

@ApplicationScoped
public class TestRoutingObservers {

    public final List<TrustScoreFullPayload> fullReceived = new CopyOnWriteArrayList<>();
    public final List<TrustScoreDeltaPayload> deltaReceived = new CopyOnWriteArrayList<>();
    public final List<TrustScoreComputedAt> notifyReceived = new CopyOnWriteArrayList<>();

    /** Reset between tests via @BeforeEach. */
    public static volatile CountDownLatch asyncLatch = new CountDownLatch(1);

    public void onFull(@Observes final TrustScoreFullPayload payload) {
        fullReceived.add(payload);
    }

    public void onDelta(@Observes final TrustScoreDeltaPayload payload) {
        deltaReceived.add(payload);
    }

    public void onNotify(@Observes final TrustScoreComputedAt payload) {
        notifyReceived.add(payload);
    }

    public CompletionStage<Void> onNotifyAsync(@ObservesAsync final TrustScoreComputedAt payload) {
        asyncLatch.countDown();
        return CompletableFuture.completedFuture(null);
    }

    public void reset() {
        fullReceived.clear();
        deltaReceived.clear();
        notifyReceived.clear();
        asyncLatch = new CountDownLatch(1);
    }
}
```

- [ ] **Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/routing/TestRoutingObservers.java
git commit -m "test: add TestRoutingObservers CDI bean for routing IT assertions

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 8: Integration tests — routing enabled (TrustScoreRoutingIT)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingIT.java`

- [ ] **Write the failing integration tests**

```java
// runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingIT.java
package io.casehub.ledger.service.routing;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(TrustScoreRoutingIT.RoutingTestProfile.class)
class TrustScoreRoutingIT {

    public static class RoutingTestProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "routing-test";
        }
    }

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    @Inject
    TestRoutingObservers observers;

    @BeforeEach
    void reset() {
        observers.reset();
    }

    // ── Happy path: full observer receives all scores ─────────────────────────

    @Test
    @Transactional
    void fullObserver_receivesAllScoresAfterComputation() {
        final String actorId = "routing-full-" + UUID.randomUUID();
        seedDecision(actorId, Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        assertThat(observers.fullReceived).hasSize(1);
        assertThat(observers.fullReceived.get(0).scores())
                .anyMatch(s -> s.actorId.equals(actorId));
    }

    // ── Happy path: notify observer receives correct actorCount ───────────────

    @Test
    @Transactional
    void notifyObserver_receivesCorrectActorCount() {
        final String actorA = "routing-notify-a-" + UUID.randomUUID();
        final String actorB = "routing-notify-b-" + UUID.randomUUID();
        seedDecision(actorA, Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);
        seedDecision(actorB, Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.FLAGGED);

        trustScoreJob.runComputation();

        assertThat(observers.notifyReceived).hasSize(1);
        assertThat(observers.notifyReceived.get(0).actorCount()).isGreaterThanOrEqualTo(2);
    }

    // ── Happy path: delta observer receives only changed actors on second run ──

    @Test
    @Transactional
    void deltaObserver_secondRun_receivesOnlyChangedActors() {
        final String stableActor = "routing-stable-" + UUID.randomUUID();
        final String changedActor = "routing-changed-" + UUID.randomUUID();

        // First run — both actors appear (first-run, previous=0.0)
        seedDecision(stableActor, Instant.now().minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND);
        seedDecision(changedActor, Instant.now().minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND);
        trustScoreJob.runComputation();
        observers.reset();

        // Second run — add a new FLAGGED attestation for changedActor only
        seedDecision(changedActor, Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.FLAGGED);
        trustScoreJob.runComputation();

        assertThat(observers.deltaReceived).hasSize(1);
        final var deltas = observers.deltaReceived.get(0).deltas();
        assertThat(deltas).anyMatch(d -> d.actorId().equals(changedActor));
        assertThat(deltas).noneMatch(d -> d.actorId().equals(stableActor));
    }

    // ── Happy path: async observer receives notification ──────────────────────

    @Test
    @Transactional
    void asyncObserver_receivesNotification() throws InterruptedException {
        seedDecision("routing-async-" + UUID.randomUUID(),
                Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        final boolean called = TestRoutingObservers.asyncLatch.await(5, TimeUnit.SECONDS);
        assertThat(called).as("Async observer must be called within 5s").isTrue();
    }

    // ── Correctness: full payload is an immutable copy ─────────────────────────

    @Test
    @Transactional
    void fullPayload_scoresListIsImmutable() {
        seedDecision("routing-immut-" + UUID.randomUUID(),
                Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        assertThat(observers.fullReceived).hasSize(1);
        final var scores = observers.fullReceived.get(0).scores();
        org.assertj.core.api.Assertions.assertThatThrownBy(
                () -> scores.add(new io.casehub.ledger.runtime.model.ActorTrustScore()))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    // ── Robustness: no entries → empty scores → observers still called ─────────

    @Test
    void noEntries_observersCalledWithEmptyScores() {
        // No entries seeded for this test — use unique profile DB
        trustScoreJob.runComputation();

        // Observers are called even with zero actors
        assertThat(observers.notifyReceived).hasSize(1);
        assertThat(observers.fullReceived).hasSize(1);
        assertThat(observers.fullReceived.get(0).scores()).isEmpty();
    }

    // ── Fixtures ──────────────────────────────────────────────────────────────

    private void seedDecision(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdict) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = decisionTime;
        repo.save(entry);

        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "routing-attestor";
        att.attestorType = ActorType.SYSTEM;
        att.verdict = verdict;
        att.confidence = 0.9;
        att.occurredAt = decisionTime.plusSeconds(60);
        em.persist(att);
    }
}
```

- [ ] **Run — expect FAIL (publisher has no observers detected yet in this CDI context)**

Actually they will pass because `TestRoutingObservers` is in the test classpath and CDI will discover it.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreRoutingIT -q
```

Expected: All tests pass. If any fail, check that `TestRoutingObservers` is on the classpath and the profile H2 URL is isolated.

- [ ] **Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingIT.java
git commit -m "test: integration tests for routing signals — enabled path

5 cases: full observer, notify actorCount, delta second run, async latch,
immutable payload, empty scores robustness.

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 9: Integration test — routing disabled (TrustScoreRoutingDisabledIT)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingDisabledIT.java`

- [ ] **Write the test**

```java
// runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingDisabledIT.java
package io.casehub.ledger.service.routing;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(TrustScoreRoutingDisabledIT.RoutingDisabledTestProfile.class)
class TrustScoreRoutingDisabledIT {

    public static class RoutingDisabledTestProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "routing-disabled-test";
        }
    }

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    @Inject
    TestRoutingObservers observers;

    @BeforeEach
    void reset() {
        observers.reset();
    }

    // ── Robustness: routing-enabled=false → observers never called ─────────────

    @Test
    @Transactional
    void routingDisabled_observersNeverCalled() {
        seedDecision("routing-off-" + UUID.randomUUID(),
                Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        assertThat(observers.fullReceived).isEmpty();
        assertThat(observers.deltaReceived).isEmpty();
        assertThat(observers.notifyReceived).isEmpty();
    }

    @Test
    @Transactional
    void routingDisabled_trustScoresStillComputed() {
        // Trust score computation is unaffected by routing-enabled flag
        final String actorId = "routing-off-score-" + UUID.randomUUID();
        seedDecision(actorId, Instant.now().minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        final var score = em.find(
                io.casehub.ledger.runtime.model.ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isGreaterThan(0.5);
    }

    // ── Fixtures ──────────────────────────────────────────────────────────────

    private void seedDecision(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdict) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = decisionTime;
        repo.save(entry);

        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "routing-attestor";
        att.attestorType = ActorType.SYSTEM;
        att.verdict = verdict;
        att.confidence = 0.9;
        att.occurredAt = decisionTime.plusSeconds(60);
        em.persist(att);
    }
}
```

- [ ] **Run — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreRoutingDisabledIT -q
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Run full test suite — verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: All tests pass (was 175 before this feature).

- [ ] **Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/routing/TrustScoreRoutingDisabledIT.java
git commit -m "test: routing disabled IT — observers registered but never called, scores still computed

Refs #IMPL_ISSUE, #EPIC"
```

---

## Task 10: E2E example module — pom.xml and Flyway migration

**Files:**
- Create: `examples/trust-score-routing/pom.xml`
- Create: `examples/trust-score-routing/src/main/resources/db/migration/V2000__task_ledger_entry.sql`
- Create: `examples/trust-score-routing/src/main/resources/application.properties`

- [ ] **Create the pom.xml** (mirrors order-processing pattern)

```xml
<?xml version="1.0"?>
<!-- examples/trust-score-routing/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <!-- Standalone POM — does NOT inherit from casehub-parent. -->

  <groupId>io.casehub.ledger.examples</groupId>
  <artifactId>casehub-ledger-example-trust-score-routing</artifactId>
  <version>0.2-SNAPSHOT</version>

  <name>CaseHub Ledger - Example: Trust Score Routing</name>
  <description>
    Runnable Quarkus application demonstrating trust score routing signals.
    Observes TrustScoreFullPayload (sync) and TrustScoreComputedAt (async)
    to maintain a ranked agent list for task assignment.
  </description>

  <properties>
    <quarkus.version>3.32.2</quarkus.version>
    <casehub-ledger.version>0.2-SNAPSHOT</casehub-ledger.version>
    <surefire-plugin.version>3.2.5</surefire-plugin.version>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${quarkus.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger</artifactId>
      <version>${casehub-ledger.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals>
              <goal>build</goal>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire-plugin.version}</version>
        <configuration>
          <systemPropertyVariables>
            <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
          </systemPropertyVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Create V2000__task_ledger_entry.sql**

```sql
-- examples/trust-score-routing/src/main/resources/db/migration/V2000__task_ledger_entry.sql
CREATE TABLE task_ledger_entry (
    id            UUID         NOT NULL,
    task_type     VARCHAR(100),
    CONSTRAINT pk_task_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_task_ledger_entry_base
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Create application.properties**

```properties
# examples/trust-score-routing/src/main/resources/application.properties
quarkus.application.name=ledger-example-trust-score-routing

quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:routingexampledb;DB_CLOSE_DELAY=-1
quarkus.datasource.username=sa
quarkus.datasource.password=

quarkus.hibernate-orm.database.generation=none

quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=classpath:db/migration

quarkus.ledger.enabled=true
quarkus.ledger.trust-score.enabled=true
quarkus.ledger.trust-score.routing-enabled=true

quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository

quarkus.log.category."io.casehub.ledger".level=DEBUG
```

- [ ] **Commit**

```bash
git add examples/trust-score-routing/
git commit -m "feat: e2e example trust-score-routing — pom, migration, application.properties

Refs #E2E_ISSUE, #EPIC"
```

---

## Task 11: E2E example module — domain and observer beans

**Files:**
- Create: `examples/trust-score-routing/src/main/java/io/casehub/ledger/examples/routing/ledger/TaskLedgerEntry.java`
- Create: `examples/trust-score-routing/src/main/java/io/casehub/ledger/examples/routing/routing/TaskRouter.java`
- Create: `examples/trust-score-routing/src/main/java/io/casehub/ledger/examples/routing/routing/RoutingSignalLogger.java`
- Create: `examples/trust-score-routing/src/main/java/io/casehub/ledger/examples/routing/api/TaskRoutingResource.java`

- [ ] **Write TaskLedgerEntry**

```java
// .../ledger/TaskLedgerEntry.java
package io.casehub.ledger.examples.routing.ledger;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;

@Entity
@Table(name = "task_ledger_entry")
public class TaskLedgerEntry extends LedgerEntry {

    @Column(name = "task_type")
    public String taskType;
}
```

- [ ] **Write TaskRouter** (sync full observer — builds ranked agent list)

```java
// .../routing/TaskRouter.java
package io.casehub.ledger.examples.routing.routing;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import io.casehub.ledger.runtime.service.routing.TrustScoreFullPayload;

@ApplicationScoped
public class TaskRouter {

    private volatile List<String> rankedAgents = List.of();

    public void onScoresUpdated(@Observes final TrustScoreFullPayload payload) {
        rankedAgents = payload.scores().stream()
                .sorted(Comparator.comparingDouble(s -> -s.trustScore))
                .map(s -> s.actorId)
                .toList();
    }

    public List<String> getRankedAgents() {
        return new ArrayList<>(rankedAgents);
    }
}
```

- [ ] **Write RoutingSignalLogger** (async notify observer — logs refresh signal)

```java
// .../routing/RoutingSignalLogger.java
package io.casehub.ledger.examples.routing.routing;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CountDownLatch;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.service.routing.TrustScoreComputedAt;

@ApplicationScoped
public class RoutingSignalLogger {

    private static final Logger log = Logger.getLogger(RoutingSignalLogger.class);

    /** Exposed for IT assertions. */
    public static volatile CountDownLatch notifyLatch = new CountDownLatch(1);

    public CompletionStage<Void> onNotification(
            @ObservesAsync final TrustScoreComputedAt notification) {
        log.infof("Trust scores refreshed at %s for %d actors",
                notification.computedAt(), notification.actorCount());
        notifyLatch.countDown();
        return CompletableFuture.completedFuture(null);
    }

    public static void resetLatch() {
        notifyLatch = new CountDownLatch(1);
    }
}
```

- [ ] **Write TaskRoutingResource**

```java
// .../api/TaskRoutingResource.java
package io.casehub.ledger.examples.routing.api;

import java.util.List;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

import io.casehub.ledger.examples.routing.routing.TaskRouter;

@Path("/routing")
@Produces(MediaType.APPLICATION_JSON)
public class TaskRoutingResource {

    @Inject
    TaskRouter taskRouter;

    @GET
    @Path("/ranked-agents")
    public List<String> rankedAgents() {
        return taskRouter.getRankedAgents();
    }
}
```

- [ ] **Compile the example module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile \
  -pl examples/trust-score-routing \
  -am -q
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add examples/trust-score-routing/src/main/java/
git commit -m "feat: trust-score-routing example — TaskRouter (sync full), RoutingSignalLogger (async notify), TaskRoutingResource

Refs #E2E_ISSUE, #EPIC"
```

---

## Task 12: E2E integration test

**Files:**
- Create: `examples/trust-score-routing/src/test/java/io/casehub/ledger/examples/routing/TrustScoreRoutingE2EIT.java`

- [ ] **Write the E2E integration test**

```java
// .../routing/TrustScoreRoutingE2EIT.java
package io.casehub.ledger.examples.routing;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.examples.routing.ledger.TaskLedgerEntry;
import io.casehub.ledger.examples.routing.routing.RoutingSignalLogger;
import io.casehub.ledger.examples.routing.routing.TaskRouter;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class TrustScoreRoutingE2EIT {

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    EntityManager em;

    @Inject
    TaskRouter taskRouter;

    @BeforeEach
    void resetLatch() {
        RoutingSignalLogger.resetLatch();
    }

    // ── Happy path: full pipeline — entries → scores → routing ────────────────

    @Test
    @Transactional
    void highTrustAgent_rankedAboveLowTrustAgent() {
        final String highTrust = "e2e-high-" + UUID.randomUUID();
        final String lowTrust = "e2e-low-" + UUID.randomUUID();

        // high-trust: 3 SOUND attestations
        seedTask(highTrust, AttestationVerdict.SOUND);
        seedTask(highTrust, AttestationVerdict.SOUND);
        seedTask(highTrust, AttestationVerdict.ENDORSED);

        // low-trust: 3 FLAGGED attestations
        seedTask(lowTrust, AttestationVerdict.FLAGGED);
        seedTask(lowTrust, AttestationVerdict.CHALLENGED);
        seedTask(lowTrust, AttestationVerdict.FLAGGED);

        trustScoreJob.runComputation();

        final List<String> ranked = taskRouter.getRankedAgents();
        assertThat(ranked.indexOf(highTrust)).isLessThan(ranked.indexOf(lowTrust));
    }

    // ── Happy path: REST endpoint returns ranked agents ───────────────────────

    @Test
    @Transactional
    void rankedAgentsEndpoint_returnsNonEmptyList() {
        seedTask("e2e-rest-" + UUID.randomUUID(), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        given()
                .when().get("/routing/ranked-agents")
                .then()
                .statusCode(200)
                .body("$", org.hamcrest.Matchers.not(org.hamcrest.Matchers.empty()));
    }

    // ── Happy path: async logger receives notification ─────────────────────────

    @Test
    @Transactional
    void asyncLogger_receivesNotificationAfterComputation() throws InterruptedException {
        seedTask("e2e-async-" + UUID.randomUUID(), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        final boolean called = RoutingSignalLogger.notifyLatch.await(5, TimeUnit.SECONDS);
        assertThat(called).as("Async notification logger must be triggered within 5s").isTrue();
    }

    // ── Correctness: ranking is stable — deterministic for same scores ─────────

    @Test
    @Transactional
    void twoRuns_rankingStableWhenScoresUnchanged() {
        final String agentA = "e2e-stable-a-" + UUID.randomUUID();
        final String agentB = "e2e-stable-b-" + UUID.randomUUID();
        seedTask(agentA, AttestationVerdict.SOUND);
        seedTask(agentA, AttestationVerdict.SOUND);
        seedTask(agentB, AttestationVerdict.SOUND);

        trustScoreJob.runComputation();
        final List<String> firstRanking = taskRouter.getRankedAgents();

        trustScoreJob.runComputation();
        final List<String> secondRanking = taskRouter.getRankedAgents();

        // Order of agents present in both runs should be the same
        final var firstFiltered = firstRanking.stream()
                .filter(a -> a.equals(agentA) || a.equals(agentB)).toList();
        final var secondFiltered = secondRanking.stream()
                .filter(a -> a.equals(agentA) || a.equals(agentB)).toList();
        assertThat(firstFiltered).isEqualTo(secondFiltered);
    }

    // ── Fixture ───────────────────────────────────────────────────────────────

    private void seedTask(final String actorId, final AttestationVerdict verdict) {
        final Instant now = Instant.now().minus(1, ChronoUnit.DAYS);

        final TaskLedgerEntry entry = new TaskLedgerEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "TaskAgent";
        entry.occurredAt = now;
        entry.taskType = "classification";
        em.persist(entry);

        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "e2e-attestor";
        att.attestorType = ActorType.SYSTEM;
        att.verdict = verdict;
        att.confidence = 0.9;
        att.occurredAt = now.plusSeconds(60);
        em.persist(att);
    }
}
```

- [ ] **Run the E2E test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -pl examples/trust-score-routing \
  -am -q
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

- [ ] **Commit**

```bash
git add examples/trust-score-routing/src/test/
git commit -m "test: E2E integration tests for trust-score-routing example

4 cases: ranking order, REST endpoint, async logger, stable ranking.

Refs #E2E_ISSUE, #EPIC"
```

---

## Task 13: Register example in root pom.xml

**Files:**
- Modify: `pom.xml` (root)

- [ ] **Add the example module to the root pom `<modules>` section**

In the root `pom.xml`, find the `<modules>` section (which currently contains `examples/order-processing`) and add the new module:

```xml
        <module>examples/trust-score-routing</module>
```

- [ ] **Build the full project to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add pom.xml
git commit -m "build: register examples/trust-score-routing in root pom

Closes #E2E_ISSUE, Refs #EPIC"
```

---

## Task 14: Fix DESIGN.md and RESEARCH.md staleness

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `docs/RESEARCH.md`

**DESIGN.md — what to change:**

1. **Near-term section** (~line 463): Remove or update the "OTel trace ID auto-wiring" paragraph — it was implemented in #31. Replace with a note that it shipped.

2. **Medium-term section** (~line 464–470): Remove the "OTel correlation wiring" entry (done). Keep "Quarkiverse submission" and "CaseHub consumer".

3. **Implementation tracker** — three changes:
   - Row "OTel correlation wiring" (⬜ Pending): change to ✅ Done, update description to: `LedgerTraceListener (@ApplicationScoped JPA entity listener), LedgerTraceIdProvider SPI, OtelTraceIdProvider (@DefaultBean). Closes #31.`
   - Add new row after the EigenTrust row:
     ```
     | **Trust score routing signals** | ✅ Done | TrustScoreRoutingPublisher, payload types (TrustScoreFullPayload, TrustScoreDeltaPayload, TrustScoreComputedAt, TrustScoreDelta), LedgerConfig.routingDeltaThreshold, TrustScoreJob wiring. CDI event dispatch, sync/async per-consumer. |
     ```

- [ ] **Apply DESIGN.md changes**

Read the current file (`docs/DESIGN.md`), locate the exact strings for the three changes, and apply targeted edits using the Edit tool. Specifically:
  - Find `| **OTel correlation wiring** | ⬜ Pending |` and change to `✅ Done` with updated description.
  - Find the medium-term section text referencing "OTel trace ID auto-wiring" as future work and update it to note it shipped in #31 (correlationId renamed to traceId, LedgerTraceListener).
  - Add the trust score routing signals row to the tracker.

**RESEARCH.md — what to change:**

1. **Priority matrix** (~line 27): Item #10 (LLM agent mesh) — change `★★★` and open status to `✅ Done`. All child issues (#23–#27) closed per DESIGN.md tracker.

2. **Priority matrix** (~line 28): Item #11 (EigenTrust transitivity) — change `★★★` open to `✅ Done`. Issue #26 closed.

3. **Priority matrix** — add row for trust score routing signals: `| 14 | **Trust score routing signals** | ~~S~~ | ~~Medium~~ | ✅ Done | CDI event dispatch after TrustScoreJob. Three payload types. Consumers select strategy by observed type. |`

4. **What's Next section** (~line 36): Update to reflect current open items — only EERP (#12) and ZK proofs (#13) remain. Items #10 and #11 are now done.

5. **Last updated date** in header: change from `2026-04-20` to `2026-04-22`.

- [ ] **Apply RESEARCH.md changes** — read the file and apply targeted edits.

- [ ] **Run full test suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q
```

Expected: All tests pass.

- [ ] **Commit**

```bash
git add docs/DESIGN.md docs/RESEARCH.md
git commit -m "docs: fix DESIGN.md and RESEARCH.md staleness

DESIGN.md: OTel correlation wiring tracker → Done (was Pending); add trust score
routing signals tracker entry; update near/medium-term sections.
RESEARCH.md: items #10 (agent mesh) and #11 (EigenTrust) → Done; add #14 routing
signals; update What's Next; bump last-updated to 2026-04-22.

Closes #DOCS_ISSUE, Closes #IMPL_ISSUE, Closes #EPIC"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ `TrustScoreFullPayload`, `TrustScoreDeltaPayload`, `TrustScoreComputedAt`, `TrustScoreDelta` — Task 1
- ✅ `TrustScoreRoutingPublisher` with `computeDeltas` — Task 3
- ✅ `routingDeltaThreshold` config key — Task 4
- ✅ `TrustScoreJob` wiring — Task 5
- ✅ Observer detection via `BeanManager.resolveObserverMethods` — Task 3
- ✅ `routing-enabled=false` fast path — Task 3 (publisher) + Task 9 (IT)
- ✅ Sync observer exception swallowed, others still called — Task 3 (publisher)
- ✅ Async observer fire-and-forget — Task 3 (publisher)
- ✅ First run: `previousScore=0.0` — Task 2 (unit test), Task 3 (computeDeltas)
- ✅ `List.copyOf()` immutability for full payload — Task 3 + Task 8 (IT)
- ✅ Delta pre-read only when delta observers registered — Task 5 (job)
- ✅ E2E example: sync full observer + async notify observer + REST endpoint — Tasks 11–12
- ✅ DESIGN.md + RESEARCH.md staleness — Task 14
- ✅ All commits linked to issues/epic — every task

**Type consistency:** `TrustScoreDelta` record fields (`actorId`, `previousScore`, `newScore`, `previousGlobalScore`, `newGlobalScore`) match across Task 1 (declaration), Task 3 (computeDeltas), Task 2 (unit tests), and Task 8 (IT assertions). `Map<String, ActorTrustScore>` used consistently for `previousSnapshot` in Tasks 3, 5, 8, 9.

**No placeholders:** All code blocks are complete and compilable.
