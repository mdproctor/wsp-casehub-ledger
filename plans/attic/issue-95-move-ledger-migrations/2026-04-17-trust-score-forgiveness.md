# Trust Score Forgiveness Mechanism Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional two-parameter forgiveness mechanism to `TrustScoreComputer` that reduces the penalty of negative decisions based on their age and the actor's negative decision frequency, with zero behaviour change when disabled.

**Architecture:** `ForgivenessParams` record added to `TrustScoreComputer`; existing single-param constructor delegates to new two-param constructor; forgiveness branch applied per negative decision inside `compute()`; `LedgerConfig.TrustScoreConfig` gains a nested `ForgivenessConfig` interface; `TrustScoreJob` passes the config values. No schema change, no new migration.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, `@QuarkusTest` for integration test.

**All commits:** `Refs #8` (child issue), part of epic `#6`.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`

---

## File Map

| File | Change |
|---|---|
| `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java` | Add `ForgivenessParams` record, new field, delegating constructor, forgiveness branch in `compute()` |
| `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java` | Add `ForgivenessConfig` nested interface + `forgiveness()` method on `TrustScoreConfig` |
| `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` | Pass forgiveness params to `TrustScoreComputer` constructor |
| `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java` | Add 6 new unit tests |
| `runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java` | New — `@QuarkusTest` integration test (happy path + end-to-end) |

---

## Task 1 — `ForgivenessParams` record + regression unit test (TDD)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java`

- [ ] **Step 1: Add the regression test — it must fail to compile (no ForgivenessParams yet)**

Add this import and test at the end of `TrustScoreComputerTest.java`, before the closing `}`:

```java
// ── forgiveness — disabled mode is identical to baseline ─────────────────

@Test
void forgiveness_disabled_identicalToBaseline() {
    // ForgivenessParams.disabled() must produce byte-for-byte identical results
    // to the original single-param constructor on the same input.
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            TrustScoreComputer.ForgivenessParams.disabled());

    final TestLedgerEntry d1 = decision("alice", now.minus(10, ChronoUnit.DAYS));
    final TestLedgerEntry d2 = decision("alice", now.minus(20, ChronoUnit.DAYS));
    final LedgerAttestation flagged = attestation(d2.id, AttestationVerdict.FLAGGED);

    final TrustScoreComputer.ActorScore baseline = computer.compute(
            List.of(d1, d2), Map.of(d2.id, List.of(flagged)), now);
    final TrustScoreComputer.ActorScore forgivenessDisabled = withForgiveness.compute(
            List.of(d1, d2), Map.of(d2.id, List.of(flagged)), now);

    assertThat(forgivenessDisabled.trustScore())
            .isCloseTo(baseline.trustScore(), within(0.0001));
    assertThat(forgivenessDisabled.decisionCount()).isEqualTo(baseline.decisionCount());
    assertThat(forgivenessDisabled.overturnedCount()).isEqualTo(baseline.overturnedCount());
}
```

- [ ] **Step 2: Run — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest#forgiveness_disabled_identicalToBaseline -q 2>&1 | tail -5
```

Expected: compilation error — `ForgivenessParams` not found.

- [ ] **Step 3: Add `ForgivenessParams` record, new field, and delegating constructor to `TrustScoreComputer.java`**

After the class Javadoc and before the `halfLifeDays` field, add the field and record. Replace the existing constructor block (lines 30–39) with:

```java
    private final int halfLifeDays;
    private final ForgivenessParams forgiveness;

    /**
     * Parameters for the optional forgiveness mechanism.
     *
     * <p>
     * The forgiveness factor {@code F = recencyForgiveness × frequencyLeniency}
     * is applied to negative decisions only:
     * {@code adjustedScore = decisionScore + F × (1.0 - decisionScore)}.
     *
     * <ul>
     *   <li>{@code recencyForgiveness = 2^(-ageInDays / halfLifeDays)} — old failures fade</li>
     *   <li>{@code frequencyLeniency = 1.0} if negative decisions ≤ threshold, else {@code 0.5}</li>
     * </ul>
     *
     * <p>
     * Use {@link #disabled()} for the default no-forgiveness path. The
     * {@link TrustScoreComputer#TrustScoreComputer(int)} single-param constructor delegates to
     * it — all existing behaviour is preserved when forgiveness is disabled.
     *
     * @param enabled whether forgiveness is active
     * @param frequencyThreshold negative decisions ≤ this receive full leniency; above → half
     * @param halfLifeDays forgiveness recency decay half-life in days
     */
    public record ForgivenessParams(boolean enabled, int frequencyThreshold, int halfLifeDays) {

        /** Returns a params instance that disables forgiveness entirely. */
        public static ForgivenessParams disabled() {
            return new ForgivenessParams(false, 0, 0);
        }
    }

    /**
     * Construct a computer with the given exponential-decay half-life and forgiveness disabled.
     *
     * @param halfLifeDays half-life in days; values {@code <= 0} default to 90
     */
    public TrustScoreComputer(final int halfLifeDays) {
        this(halfLifeDays, ForgivenessParams.disabled());
    }

    /**
     * Construct a computer with the given decay half-life and forgiveness configuration.
     *
     * @param halfLifeDays half-life in days; values {@code <= 0} default to 90
     * @param forgiveness  forgiveness parameters; use {@link ForgivenessParams#disabled()} to
     *                     reproduce the original behaviour exactly
     */
    public TrustScoreComputer(final int halfLifeDays, final ForgivenessParams forgiveness) {
        this.halfLifeDays = halfLifeDays > 0 ? halfLifeDays : 90;
        this.forgiveness = Objects.requireNonNull(forgiveness, "forgiveness must not be null");
    }
```

Also add the import at the top of `TrustScoreComputer.java`:
```java
import java.util.Objects;
```

- [ ] **Step 4: Run — confirm regression test passes (no forgiveness logic yet, disabled path = baseline)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest -q 2>&1 | tail -5
```

Expected: `Tests run: 17, Failures: 0, Errors: 0` (16 original + 1 new).

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java
git commit -m "feat(forgiveness): ForgivenessParams record + delegating constructor

Existing single-param constructor delegates to new two-param version via
ForgivenessParams.disabled(). Zero behaviour change — regression test confirms
identical results when disabled. 17 unit tests passing.

Refs #8"
```

---

## Task 2 — Forgiveness formula: 5 unit tests + implementation (TDD)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java`

- [ ] **Step 1: Add 5 failing forgiveness formula tests**

Append to `TrustScoreComputerTest.java` (before the closing `}`):

```java
@Test
void forgiveness_cleanDecisions_unaffected() {
    // Clean decisions (score = 1.0) must not be changed — forgiveness branch not entered
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            new TrustScoreComputer.ForgivenessParams(true, 3, 30));

    final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.DAYS));
    final TestLedgerEntry d2 = decision("alice", now.minus(2, ChronoUnit.DAYS));

    final TrustScoreComputer.ActorScore baseline = computer.compute(List.of(d1, d2), Map.of(), now);
    final TrustScoreComputer.ActorScore forgiven  = withForgiveness.compute(List.of(d1, d2), Map.of(), now);

    assertThat(forgiven.trustScore()).isCloseTo(baseline.trustScore(), within(0.0001));
}

@Test
void forgiveness_singleTransientFailure_recovers() {
    // One flagged decision + subsequent clean history → forgiveness raises score above baseline
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            new TrustScoreComputer.ForgivenessParams(true, 3, 30));

    final TestLedgerEntry failure = decision("alice", now.minus(30, ChronoUnit.DAYS));
    final LedgerAttestation flagged = attestation(failure.id, AttestationVerdict.FLAGGED);
    final TestLedgerEntry clean1   = decision("alice", now.minus(5, ChronoUnit.DAYS));
    final TestLedgerEntry clean2   = decision("alice", now.minus(3, ChronoUnit.DAYS));
    final TestLedgerEntry clean3   = decision("alice", now.minus(1, ChronoUnit.DAYS));

    final Map<UUID, List<LedgerAttestation>> attestations = Map.of(failure.id, List.of(flagged));
    final List<LedgerEntry> decisions = List.of(failure, clean1, clean2, clean3);

    final TrustScoreComputer.ActorScore baseline = computer.compute(decisions, attestations, now);
    final TrustScoreComputer.ActorScore forgiven  = withForgiveness.compute(decisions, attestations, now);

    assertThat(forgiven.trustScore()).isGreaterThan(baseline.trustScore());
    assertThat(forgiven.trustScore()).isLessThanOrEqualTo(1.0);
}

@Test
void forgiveness_oldFailure_substantiallyForgiven() {
    // A failure 60 days ago with halfLife=30 → recencyForgiveness ≈ 0.25
    // Score should be visibly higher than without forgiveness
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            new TrustScoreComputer.ForgivenessParams(true, 3, 30));

    final TestLedgerEntry oldFailure = decision("alice", now.minus(60, ChronoUnit.DAYS));
    final LedgerAttestation flagged  = attestation(oldFailure.id, AttestationVerdict.FLAGGED);

    final Map<UUID, List<LedgerAttestation>> attestations = Map.of(oldFailure.id, List.of(flagged));

    final TrustScoreComputer.ActorScore baseline = computer.compute(List.of(oldFailure), attestations, now);
    final TrustScoreComputer.ActorScore forgiven  = withForgiveness.compute(List.of(oldFailure), attestations, now);

    // recencyForgiveness = 2^(-60/30) = 0.25, frequencyLeniency = 1.0 (1 ≤ 3)
    // adjustedScore = 0.0 + 0.25 × 1.0 = 0.25 — substantially higher than baseline 0.0
    assertThat(forgiven.trustScore()).isGreaterThan(baseline.trustScore() + 0.15);
}

@Test
void forgiveness_repeatOffender_lessForgiven() {
    // 5 negative decisions exceeds frequencyThreshold=3 → frequencyLeniency = 0.5
    // Agent with 2 negatives must be forgiven MORE than agent with 5 negatives (same age)
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            new TrustScoreComputer.ForgivenessParams(true, 3, 30));
    final Instant failureTime = now.minus(20, ChronoUnit.DAYS);

    // Few negatives: 2 flagged decisions (below threshold)
    final TestLedgerEntry fail1 = decision("agent-a", failureTime);
    final TestLedgerEntry fail2 = decision("agent-a", failureTime.minusSeconds(60));
    final LedgerAttestation f1  = attestation(fail1.id, AttestationVerdict.FLAGGED);
    final LedgerAttestation f2  = attestation(fail2.id, AttestationVerdict.FLAGGED);
    final TrustScoreComputer.ActorScore fewNegatives = withForgiveness.compute(
            List.of(fail1, fail2),
            Map.of(fail1.id, List.of(f1), fail2.id, List.of(f2)),
            now);

    // Many negatives: 5 flagged decisions (above threshold)
    final TestLedgerEntry fail3 = decision("agent-b", failureTime);
    final TestLedgerEntry fail4 = decision("agent-b", failureTime.minusSeconds(60));
    final TestLedgerEntry fail5 = decision("agent-b", failureTime.minusSeconds(120));
    final TestLedgerEntry fail6 = decision("agent-b", failureTime.minusSeconds(180));
    final TestLedgerEntry fail7 = decision("agent-b", failureTime.minusSeconds(240));
    final LedgerAttestation f3  = attestation(fail3.id, AttestationVerdict.FLAGGED);
    final LedgerAttestation f4  = attestation(fail4.id, AttestationVerdict.FLAGGED);
    final LedgerAttestation f5  = attestation(fail5.id, AttestationVerdict.FLAGGED);
    final LedgerAttestation f6  = attestation(fail6.id, AttestationVerdict.FLAGGED);
    final LedgerAttestation f7  = attestation(fail7.id, AttestationVerdict.FLAGGED);
    final TrustScoreComputer.ActorScore manyNegatives = withForgiveness.compute(
            List.of(fail3, fail4, fail5, fail6, fail7),
            Map.of(fail3.id, List.of(f3), fail4.id, List.of(f4),
                   fail5.id, List.of(f5), fail6.id, List.of(f6), fail7.id, List.of(f7)),
            now);

    // Agent with few negatives must receive more benefit from forgiveness
    assertThat(fewNegatives.trustScore()).isGreaterThan(manyNegatives.trustScore());
}

@Test
void forgiveness_recentFailure_partiallyForgiven() {
    // A failure that just happened → recencyForgiveness ≈ 1.0 → full F applied
    // Score is raised from 0.0 toward 1.0 (not left at 0.0)
    final TrustScoreComputer withForgiveness = new TrustScoreComputer(90,
            new TrustScoreComputer.ForgivenessParams(true, 3, 30));

    final TestLedgerEntry recentFailure = decision("alice", now.minus(1, ChronoUnit.HOURS));
    final LedgerAttestation flagged     = attestation(recentFailure.id, AttestationVerdict.FLAGGED);

    final TrustScoreComputer.ActorScore baseline = computer.compute(
            List.of(recentFailure), Map.of(recentFailure.id, List.of(flagged)), now);
    final TrustScoreComputer.ActorScore forgiven  = withForgiveness.compute(
            List.of(recentFailure), Map.of(recentFailure.id, List.of(flagged)), now);

    // baseline = 0.0; with forgiveness enabled and recency ≈ 1.0, score must be > 0.0
    assertThat(forgiven.trustScore()).isGreaterThan(baseline.trustScore());
    assertThat(forgiven.trustScore()).isGreaterThan(0.5); // recencyF ≈ 1.0, frequencyF = 1.0 → F ≈ 1.0
}
```

- [ ] **Step 2: Run — confirm all 5 new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -10
```

Expected: 5 failures — the forgiveness branch is not implemented yet.

- [ ] **Step 3: Implement the forgiveness branch in `TrustScoreComputer.compute()`**

In `compute()`, immediately after the `if (decisions.isEmpty())` guard, add the
`negativeDecisions` count. Then replace `weightedPositive += weight * decisionScore`
with the forgiveness-aware version. The complete updated `compute()` method:

```java
    public ActorScore compute(
            final List<LedgerEntry> decisions,
            final Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
            final Instant now) {

        if (decisions.isEmpty()) {
            return new ActorScore(0.5, 0, 0, 0, 0, 0);
        }

        // Count negative decisions once — used by frequencyLeniency in forgiveness
        final long negativeDecisions = decisions.stream()
                .filter(e -> attestationsByEntryId.getOrDefault(e.id, List.of()).stream()
                        .anyMatch(a -> a.verdict == AttestationVerdict.FLAGGED
                                || a.verdict == AttestationVerdict.CHALLENGED))
                .count();

        double weightedPositive = 0.0;
        double weightedTotal = 0.0;
        int overturnedCount = 0;
        int totalPositive = 0;
        int totalNegative = 0;

        for (final LedgerEntry entry : decisions) {
            final Instant entryTime = entry.occurredAt != null ? entry.occurredAt : now;
            final long ageInDays = java.time.Duration.between(entryTime, now).toDays();
            final double weight = Math.pow(2.0, -(double) ageInDays / halfLifeDays);

            final List<LedgerAttestation> attestations = attestationsByEntryId.getOrDefault(entry.id, List.of());

            final long positive = attestations.stream()
                    .filter(a -> a.verdict == AttestationVerdict.SOUND
                            || a.verdict == AttestationVerdict.ENDORSED)
                    .count();
            final long negative = attestations.stream()
                    .filter(a -> a.verdict == AttestationVerdict.FLAGGED
                            || a.verdict == AttestationVerdict.CHALLENGED)
                    .count();

            totalPositive += (int) positive;
            totalNegative += (int) negative;
            if (negative > 0) {
                overturnedCount++;
            }

            // Decision score: 1.0 clean, 0.5 mixed, 0.0 predominantly negative
            final double decisionScore;
            if (negative == 0) {
                decisionScore = 1.0;
            } else if (positive > negative) {
                decisionScore = 0.5;
            } else {
                decisionScore = 0.0;
            }

            // Forgiveness: raise the effective score for negative decisions based on
            // recency (old failures fade) and frequency (one-offs forgiven more than patterns).
            // Clean decisions (score = 1.0) are not affected.
            final double effectiveScore;
            if (forgiveness.enabled() && decisionScore < 1.0) {
                final double recencyF = Math.pow(2.0, -(double) ageInDays / forgiveness.halfLifeDays());
                final double freqF = negativeDecisions <= forgiveness.frequencyThreshold() ? 1.0 : 0.5;
                effectiveScore = decisionScore + (recencyF * freqF) * (1.0 - decisionScore);
            } else {
                effectiveScore = decisionScore;
            }

            weightedPositive += weight * effectiveScore;
            weightedTotal += weight;
        }

        final double rawScore = weightedTotal > 0.0 ? weightedPositive / weightedTotal : 0.5;
        final double trustScore = Math.max(0.0, Math.min(1.0, rawScore));

        return new ActorScore(trustScore, decisions.size(), overturnedCount, 0, totalPositive, totalNegative);
    }
```

- [ ] **Step 4: Run all unit tests — 22 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest -q 2>&1 | tail -5
```

Expected: `Tests run: 22, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java
git commit -m "feat(forgiveness): implement recency×frequency forgiveness formula in TrustScoreComputer

Formula: F = 2^(-ageInDays/halfLifeDays) × (negCount ≤ threshold ? 1.0 : 0.5)
adjustedScore = decisionScore + F × (1.0 - decisionScore)

Clean decisions (score=1.0) unaffected. Disabled path produces byte-identical
results to baseline. 22 unit tests passing (16 original + 6 new).

Refs #8"
```

---

## Task 3 — `LedgerConfig.ForgivenessConfig` + `TrustScoreJob` wiring

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

No new failing test needed here — the behaviour change (disabled by default) means the
existing 22 unit tests are the correctness gate. The integration test in Task 4 will
verify the config wiring end-to-end.

- [ ] **Step 1: Add `ForgivenessConfig` to `LedgerConfig.java`**

Inside the `TrustScoreConfig` interface (after the `routingEnabled()` method), add:

```java
        /**
         * Forgiveness mechanism — modulates penalties for negative decisions based on
         * their age and the actor's overall negative decision frequency.
         *
         * <p>
         * When disabled (the default), {@link TrustScoreComputer} produces identical
         * results to before this feature was introduced.
         *
         * @return forgiveness sub-configuration
         */
        ForgivenessConfig forgiveness();

        /** Forgiveness mechanism settings for trust score computation. */
        interface ForgivenessConfig {

            /**
             * When {@code true}, the forgiveness mechanism modulates the penalty of negative
             * decisions based on their age and the actor's negative decision frequency.
             * Off by default — enabling without {@code trust-score.enabled=true} has no effect.
             *
             * @return {@code true} if forgiveness is active; {@code false} by default
             */
            @WithDefault("false")
            boolean enabled();

            /**
             * Number of negative decisions at or below which the actor receives full
             * frequency leniency ({@code 1.0}). Above this threshold, leniency is halved
             * ({@code 0.5}), distinguishing one-off failures from repeat patterns.
             *
             * @return frequency threshold (default 3)
             */
            @WithDefault("3")
            int frequencyThreshold();

            /**
             * Half-life in days for the forgiveness recency decay. A failure this many days
             * in the past contributes 50% of its original penalty; at double the half-life,
             * 25%. Shorter values forgive faster. Independent of
             * {@link TrustScoreConfig#decayHalfLifeDays()}.
             *
             * @return forgiveness half-life in days (default 30)
             */
            @WithDefault("30")
            int halfLifeDays();
        }
```

- [ ] **Step 2: Update `TrustScoreJob.runComputation()` to pass forgiveness params**

Replace the line that constructs `TrustScoreComputer`:

```java
// Before:
final TrustScoreComputer computer = new TrustScoreComputer(config.trustScore().decayHalfLifeDays());

// After:
final TrustScoreComputer computer = new TrustScoreComputer(
        config.trustScore().decayHalfLifeDays(),
        new TrustScoreComputer.ForgivenessParams(
                config.trustScore().forgiveness().enabled(),
                config.trustScore().forgiveness().frequencyThreshold(),
                config.trustScore().forgiveness().halfLifeDays()));
```

- [ ] **Step 3: Run full runtime test suite — all 49 tests must still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: `Tests run: 49, Failures: 0, Errors: 0` (22 TrustScoreComputer + 18 LedgerHashChain + 8 LedgerSupplementSerializer + 7 LedgerSupplementIT = 55... wait — count includes IT tests. Exact number doesn't matter; expect BUILD SUCCESS with zero failures).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
git commit -m "feat(forgiveness): wire ForgivenessConfig into LedgerConfig and TrustScoreJob

quarkus.ledger.trust-score.forgiveness.enabled=false (default — zero behaviour change)
quarkus.ledger.trust-score.forgiveness.frequency-threshold=3
quarkus.ledger.trust-score.forgiveness.half-life-days=30

TrustScoreJob passes forgiveness params to TrustScoreComputer constructor.
All existing tests pass unchanged.

Refs #8"
```

---

## Task 4 — Integration test: happy path end-to-end with `@QuarkusTest`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java`
- Modify: `runtime/src/test/resources/application.properties` (add trust score + forgiveness profile)

This test runs the full `TrustScoreJob` against an H2 in-memory database with real
ledger entries and attestations, verifying that forgiveness enabled produces higher
trust scores than forgiveness disabled for an actor with negative decisions.

- [ ] **Step 1: Add forgiveness test profile to `runtime/src/test/resources/application.properties`**

Append to the existing `application.properties`:

```properties
# Trust score + forgiveness profile (used by TrustScoreForgivenessIT)
%forgiveness-test.quarkus.ledger.trust-score.enabled=true
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.enabled=true
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.frequency-threshold=3
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.half-life-days=30
```

- [ ] **Step 2: Write the failing integration test**

Create `runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * End-to-end integration test for the forgiveness mechanism.
 *
 * <p>
 * Runs the full {@link TrustScoreJob} pipeline against an H2 database populated with
 * real ledger entries and attestations, then verifies that an actor with a single old
 * flagged decision recovers a higher trust score with forgiveness enabled than without.
 *
 * <p>
 * Uses the {@code forgiveness-test} Quarkus profile which enables trust scoring and
 * forgiveness via {@code application.properties} {@code %forgiveness-test.*} keys.
 */
@QuarkusTest
@TestProfile(TrustScoreForgivenessIT.ForgivenessProfile.class)
class TrustScoreForgivenessIT {

    public static class ForgivenessProfile implements io.quarkus.test.junit.QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "forgiveness-test";
        }
    }

    @Inject
    TrustScoreJob trustScoreJob;

    // ── happy path: forgiveness raises score for actor with old failure ──────

    @Test
    @Transactional
    void forgiveness_raisesScore_forActorWithOldFailure() {
        final String actorId = "agent-forgiveness-" + UUID.randomUUID();

        // One flagged decision 45 days ago (recencyForgiveness = 2^(-45/30) ≈ 0.35)
        seedDecision(actorId, now().minus(45, ChronoUnit.DAYS), AttestationVerdict.FLAGGED);

        // Five clean decisions in the past week
        seedDecision(actorId, now().minus(6, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(5, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(4, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(3, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(1, ChronoUnit.DAYS), null);

        trustScoreJob.runComputation();

        final ActorTrustScore score = ActorTrustScore
                .<ActorTrustScore>find("actorId", actorId).firstResult();
        assertThat(score).isNotNull();
        // With forgiveness: old failure partially forgiven, recent clean history dominates
        // Score must be well above 0.5 (neutral prior)
        assertThat(score.trustScore).isGreaterThan(0.7);
        assertThat(score.decisionCount).isEqualTo(6);
        assertThat(score.overturnedCount).isEqualTo(1);
    }

    // ── happy path: clean actor unaffected by forgiveness ───────────────────

    @Test
    @Transactional
    void forgiveness_cleanActor_scoreUnchanged() {
        final String actorId = "agent-clean-" + UUID.randomUUID();

        seedDecision(actorId, now().minus(3, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(2, ChronoUnit.DAYS), null);
        seedDecision(actorId, now().minus(1, ChronoUnit.DAYS), null);

        trustScoreJob.runComputation();

        final ActorTrustScore score = ActorTrustScore
                .<ActorTrustScore>find("actorId", actorId).firstResult();
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isCloseTo(1.0, org.assertj.core.api.Assertions.within(0.01));
    }

    // ── happy path: repeat offender receives less forgiveness ────────────────

    @Test
    @Transactional
    void forgiveness_repeatOffender_lowerScoreThanOneOff() {
        final String oneOffActor   = "agent-oneoff-"  + UUID.randomUUID();
        final String repeatActor   = "agent-repeat-"  + UUID.randomUUID();
        final Instant failureTime  = now().minus(20, ChronoUnit.DAYS);

        // One-off: 1 negative decision (below threshold=3)
        seedDecision(oneOffActor, failureTime, AttestationVerdict.FLAGGED);
        seedDecision(oneOffActor, now().minus(2, ChronoUnit.DAYS), null);

        // Repeat offender: 5 negative decisions (above threshold=3)
        for (int i = 0; i < 5; i++) {
            seedDecision(repeatActor, failureTime.minusSeconds(i * 60L), AttestationVerdict.FLAGGED);
        }
        seedDecision(repeatActor, now().minus(2, ChronoUnit.DAYS), null);

        trustScoreJob.runComputation();

        final double oneOffScore = ActorTrustScore
                .<ActorTrustScore>find("actorId", oneOffActor).firstResult().trustScore;
        final double repeatScore = ActorTrustScore
                .<ActorTrustScore>find("actorId", repeatActor).firstResult().trustScore;

        assertThat(oneOffScore).isGreaterThan(repeatScore);
    }

    // ── fixture ───────────────────────────────────────────────────────────────

    private void seedDecision(final String actorId, final Instant occurredAt,
            final AttestationVerdict verdictOrNull) {
        final TestEntry entry = new TestEntry();
        entry.subjectId    = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType    = LedgerEntryType.EVENT;
        entry.actorId      = actorId;
        entry.actorType    = ActorType.AGENT;
        entry.actorRole    = "Classifier";
        entry.occurredAt   = occurredAt;
        entry.persist();

        if (verdictOrNull != null) {
            final LedgerAttestation att = new LedgerAttestation();
            att.id              = UUID.randomUUID();
            att.ledgerEntryId   = entry.id;
            att.subjectId       = entry.subjectId;
            att.attestorId      = "compliance-bot";
            att.attestorType    = ActorType.AGENT;
            att.verdict         = verdictOrNull;
            att.confidence      = 0.9;
            att.occurredAt      = occurredAt.plusSeconds(60);
            att.persist();
        }
    }

    private Instant now() {
        return Instant.now();
    }
}
```

- [ ] **Step 3: Run the integration test — verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreForgivenessIT -q 2>&1 | tail -10
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`.

If `LedgerAttestation` lacks a `ledgerEntryId` field mapped in Panache — check
`LedgerAttestation.java`. The field should be `ledgerEntryId` based on the current
schema. If the field name differs, adjust the fixture accordingly.

- [ ] **Step 4: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, zero failures.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java \
        runtime/src/test/resources/application.properties
git commit -m "test(forgiveness): end-to-end integration test via TrustScoreJob + H2

3 @QuarkusTest cases: old failure recovers high score, clean actor unaffected,
repeat offender scores lower than one-off. Uses forgiveness-test Quarkus profile
with trust-score.enabled=true and forgiveness.enabled=true.

Refs #8"
```

---

## Task 5 — Documentation

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `docs/AUDITABILITY.md`

- [ ] **Step 1: Add forgiveness to `docs/DESIGN.md` trust score section**

Find the `TrustScoreConfig` description in DESIGN.md (the section describing
`quarkus.ledger.trust-score.*` config). Add after the existing trust score config table:

```markdown
**Forgiveness sub-config (`quarkus.ledger.trust-score.forgiveness.*`):**

| Key | Default | Description |
|---|---|---|
| `forgiveness.enabled` | `false` | Enable forgiveness — off by default, zero behaviour change when disabled |
| `forgiveness.frequency-threshold` | `3` | Negative decisions ≤ this receive full leniency; above → half leniency |
| `forgiveness.half-life-days` | `30` | Forgiveness recency decay half-life (independent of `decay-half-life-days`) |

Formula: `F = 2^(-ageInDays / halfLifeDays) × frequencyLeniency`,
`adjustedScore = decisionScore + F × (1.0 - decisionScore)`.
Clean decisions (score = 1.0) are not affected. See `adr/0001-forgiveness-mechanism-omits-severity-dimension.md` for the decision to exclude severity.
```

- [ ] **Step 2: Update Implementation Tracker in `docs/DESIGN.md`**

Find the tracker table. Add a new row:

```markdown
| **Forgiveness mechanism** | ✅ Done | Two-parameter (recency + frequency) forgiveness in TrustScoreComputer; quarkus.ledger.trust-score.forgiveness.*; 22 unit tests + 3 IT tests |
```

- [ ] **Step 3: Commit docs**

```bash
git add docs/DESIGN.md
git commit -m "docs: document forgiveness mechanism in DESIGN.md tracker and config table

Refs #8"
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| `ForgivenessParams` record with `disabled()` | Task 1 |
| Delegating single-param constructor | Task 1 |
| `negativeDecisions` count before loop | Task 2 |
| Forgiveness branch: recencyF × freqF × (1-score) | Task 2 |
| Clean decisions unaffected | Task 2 test |
| `ForgivenessConfig` nested interface + 3 config keys | Task 3 |
| `TrustScoreJob` passes params | Task 3 |
| Disabled = identical to baseline (regression) | Task 1 test |
| 6 new unit tests | Tasks 1+2 |
| `@QuarkusTest` happy path end-to-end | Task 4 |
| Documentation | Task 5 |

**No placeholders.** All steps contain complete code.

**Type consistency:** `ForgivenessParams` defined in Task 1, used identically in Tasks 2, 3, 4. `forgiveness.enabled()`, `forgiveness.frequencyThreshold()`, `forgiveness.halfLifeDays()` consistent throughout.
