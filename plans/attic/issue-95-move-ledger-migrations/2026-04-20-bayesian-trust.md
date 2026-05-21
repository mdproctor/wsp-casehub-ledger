# Bayesian Beta Trust Scoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `ForgivenessParams`-based weighted-average trust scoring with a Bayesian Beta distribution model where each attestation incrementally updates α/β with a recency-decayed contribution.

**Architecture:** `TrustScoreComputer.compute()` iterates over all attestations across all decisions, accumulating α (positive verdicts) and β (negative verdicts) with `recencyWeight = 2^(-ageInDays / halfLifeDays)` using the attestation's own `occurredAt`. Final score = α/(α+β). Prior is Beta(1,1) → score 0.5 with no history. `ForgivenessParams` is removed entirely — the Beta model supersedes it. `ActorScore` gains `alpha`/`beta` fields and drops `appealCount`. Downstream: `ActorTrustScore` entity, V1001 schema, repository SPI/impl, `TrustScoreJob`, and `LedgerConfig` all updated in lockstep.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, H2 (tests), `@QuarkusTest`

**Issue:** All commits reference `Refs #28`

---

## File Map

| File | Change |
|---|---|
| `runtime/src/main/java/.../service/TrustScoreComputer.java` | Full rewrite — Beta algorithm, remove `ForgivenessParams`, update `ActorScore` |
| `runtime/src/main/java/.../model/ActorTrustScore.java` | Drop `appealCount`, add `alpha`/`beta` (DOUBLE) |
| `runtime/src/main/java/.../repository/ActorTrustScoreRepository.java` | Update `upsert()` — drop `appealCount`, add `alpha`/`beta` |
| `runtime/src/main/java/.../repository/jpa/JpaActorTrustScoreRepository.java` | Update `upsert()` impl |
| `runtime/src/main/java/.../service/TrustScoreJob.java` | Single-arg constructor, updated `upsert()` call |
| `runtime/src/main/java/.../config/LedgerConfig.java` | Remove `ForgivenessConfig` + `forgiveness()` accessor |
| `runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql` | Drop `appeal_count`, add `alpha_value`/`beta_value` DOUBLE |
| `runtime/src/test/java/.../service/TrustScoreComputerTest.java` | Full rewrite for Beta model |
| `runtime/src/test/java/.../service/TrustScoreForgivenessIT.java` | **Delete** |
| `runtime/src/test/java/.../service/TrustScoreIT.java` | **Create** — Beta integration tests |
| `runtime/src/test/resources/application.properties` | Remove `forgiveness-test` profile, add `trust-score-test` profile |
| `adr/0001-forgiveness-mechanism-omits-severity-dimension.md` | Mark superseded by ADR 0003 |
| `adr/0003-bayesian-beta-trust-model.md` | **Create** — decision record |
| `docs/DESIGN.md` | Update trust scoring section, remove forgiveness config table |
| `docs/RESEARCH.md` | Mark priority #6 done |

---

## Task 1: Failing unit tests — expose the Beta behavioral gap

These 4 tests will **fail** with the current weighted-average algorithm and **pass** after the Beta rewrite in Task 2.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java`

- [ ] **Step 1: Add 4 failing tests at the end of `TrustScoreComputerTest`**

Append inside the class (after the existing `zeroHalfLifeDays_defaultsTo90` test):

```java
// ── Beta model: these tests FAIL with the current algorithm ──────────────

@Test
void beta_unattestedDecision_scoresNeutral() {
    // Current model: unattempted decisions score 1.0 (clean).
    // Beta model: unattested decisions contribute nothing — prior gives 0.5.
    final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));

    final TrustScoreComputer.ActorScore score = computer.compute(List.of(d), Map.of(), now);

    assertThat(score.trustScore()).isCloseTo(0.5, within(0.01));
}

@Test
void beta_onePositiveAttestation_scoreNotAtMax() {
    // Current model: 1 positive → score 1.0 (maximum confidence).
    // Beta model: 1 positive → α=2, β=1 → score ≈ 0.667 (uncertainty acknowledged).
    final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
    final LedgerAttestation a = attestation(d.id, AttestationVerdict.SOUND);
    a.occurredAt = now.minus(1, ChronoUnit.HOURS);

    final TrustScoreComputer.ActorScore score = computer.compute(
            List.of(d), Map.of(d.id, List.of(a)), now);

    assertThat(score.trustScore()).isLessThan(0.9); // Beta: ~0.667, not 1.0
}

@Test
void beta_oneNegativeAttestation_scoreAboveZero() {
    // Current model: 1 negative → score 0.0 (harshest penalty).
    // Beta model: 1 negative → α=1, β=2 → score ≈ 0.333 (uncertainty acknowledged).
    final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
    final LedgerAttestation a = attestation(d.id, AttestationVerdict.FLAGGED);
    a.occurredAt = now.minus(1, ChronoUnit.HOURS);

    final TrustScoreComputer.ActorScore score = computer.compute(
            List.of(d), Map.of(d.id, List.of(a)), now);

    assertThat(score.trustScore()).isGreaterThan(0.1); // Beta: ~0.333, not 0.0
}

@Test
void beta_moreEvidenceYieldsHigherConfidence() {
    // Current model: 1 positive and 100 positives both score 1.0.
    // Beta model: 1 positive → 0.667; 100 positives → 0.990. Evidence matters.
    final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.HOURS));
    final LedgerAttestation onePositive = attestation(d1.id, AttestationVerdict.SOUND);
    onePositive.occurredAt = now.minus(1, ChronoUnit.HOURS);

    final TestLedgerEntry d2 = decision("bob", now.minus(1, ChronoUnit.HOURS));
    final java.util.List<LedgerAttestation> manyPositive = new java.util.ArrayList<>();
    for (int i = 0; i < 100; i++) {
        final LedgerAttestation a = attestation(d2.id, AttestationVerdict.SOUND);
        a.occurredAt = now.minus(1, ChronoUnit.HOURS);
        manyPositive.add(a);
    }

    final TrustScoreComputer.ActorScore scoreOne = computer.compute(
            List.of(d1), Map.of(d1.id, List.of(onePositive)), now);
    final TrustScoreComputer.ActorScore scoreMany = computer.compute(
            List.of(d2), Map.of(d2.id, manyPositive), now);

    assertThat(scoreMany.trustScore()).isGreaterThan(scoreOne.trustScore());
}
```

- [ ] **Step 2: Run tests — verify exactly 4 fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreComputerTest -q 2>&1 | grep -E "FAIL|Tests run:|BUILD"
```

Expected: `Tests run: 26, Failures: 4` (the 4 new tests fail, 22 existing pass).

---

## Task 2: Rewrite `TrustScoreComputer` — Beta algorithm, remove `ForgivenessParams`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java`

- [ ] **Step 1: Replace `TrustScoreComputer.java` in full**

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Computes Bayesian Beta trust scores from ledger attestation history.
 *
 * <p>
 * Pure Java — no CDI, no database. Suitable for unit tests without a Quarkus runtime.
 *
 * <p>
 * Algorithm: start with prior Beta(1, 1). For each attestation across all of an actor's
 * decisions, compute {@code recencyWeight = 2^(-ageInDays / halfLifeDays)} using the
 * attestation's own {@code occurredAt}. SOUND/ENDORSED increments α; FLAGGED/CHALLENGED
 * increments β. Score = α/(α+β), clamped to [0.0, 1.0].
 *
 * <p>
 * Properties: no history → 0.5 (maximum uncertainty). Unattested decisions contribute
 * nothing — they do not inflate the score. More evidence yields higher confidence:
 * 1 positive → ≈0.667; 100 positives → ≈0.990. Old negative attestations fade naturally
 * via recency weighting on β.
 */
public final class TrustScoreComputer {

    private final int halfLifeDays;

    /**
     * @param halfLifeDays recency decay half-life in days; values ≤ 0 default to 90
     */
    public TrustScoreComputer(final int halfLifeDays) {
        this.halfLifeDays = halfLifeDays > 0 ? halfLifeDays : 90;
    }

    /**
     * The computed score and metrics for one actor.
     *
     * @param trustScore computed trust score in [0.0, 1.0]
     * @param alpha final α value (prior 1.0 + positive recency-weighted contributions)
     * @param beta final β value (prior 1.0 + negative recency-weighted contributions)
     * @param decisionCount number of EVENT entries evaluated
     * @param overturnedCount number of decisions with at least one negative attestation
     * @param attestationPositive total positive attestation count
     * @param attestationNegative total negative attestation count
     */
    public record ActorScore(
            double trustScore,
            double alpha,
            double beta,
            int decisionCount,
            int overturnedCount,
            int attestationPositive,
            int attestationNegative) {
    }

    /**
     * Compute a Bayesian Beta trust score for one actor.
     *
     * @param decisions EVENT ledger entries where this actor was the decision-maker
     * @param attestationsByEntryId map from entry id to its attestations
     * @param now reference timestamp for age calculation
     * @return the computed score and metrics
     */
    public ActorScore compute(
            final List<LedgerEntry> decisions,
            final Map<UUID, List<LedgerAttestation>> attestationsByEntryId,
            final Instant now) {

        if (decisions.isEmpty()) {
            return new ActorScore(0.5, 1.0, 1.0, 0, 0, 0, 0);
        }

        double alpha = 1.0;
        double beta = 1.0;
        int overturnedCount = 0;
        int totalPositive = 0;
        int totalNegative = 0;

        for (final LedgerEntry entry : decisions) {
            final List<LedgerAttestation> attestations =
                    attestationsByEntryId.getOrDefault(entry.id, List.of());
            boolean hasNegative = false;

            for (final LedgerAttestation attestation : attestations) {
                final Instant attestationTime =
                        attestation.occurredAt != null ? attestation.occurredAt : now;
                final long ageInDays =
                        java.time.Duration.between(attestationTime, now).toDays();
                final double recencyWeight =
                        Math.pow(2.0, -(double) ageInDays / halfLifeDays);

                if (attestation.verdict == AttestationVerdict.SOUND
                        || attestation.verdict == AttestationVerdict.ENDORSED) {
                    alpha += recencyWeight;
                    totalPositive++;
                } else if (attestation.verdict == AttestationVerdict.FLAGGED
                        || attestation.verdict == AttestationVerdict.CHALLENGED) {
                    beta += recencyWeight;
                    totalNegative++;
                    hasNegative = true;
                }
            }
            if (hasNegative) {
                overturnedCount++;
            }
        }

        final double rawScore = alpha / (alpha + beta);
        final double trustScore = Math.max(0.0, Math.min(1.0, rawScore));

        return new ActorScore(
                trustScore, alpha, beta,
                decisions.size(), overturnedCount,
                totalPositive, totalNegative);
    }
}
```

- [ ] **Step 2: Replace `TrustScoreComputerTest.java` in full**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.TrustScoreComputer;

/**
 * Pure JUnit 5 unit tests for {@link TrustScoreComputer} — no Quarkus runtime, no CDI.
 *
 * <p>
 * Bayesian Beta model: prior Beta(1,1). Each attestation contributes a recency-weighted
 * increment to α (positive) or β (negative). Score = α/(α+β). Unattested decisions
 * contribute nothing — only the prior applies.
 */
class TrustScoreComputerTest {

    private final TrustScoreComputer computer = new TrustScoreComputer(90);
    private final Instant now = Instant.now();

    // ── Concrete subclass (LedgerEntry is abstract) ───────────────────────────

    private static class TestLedgerEntry extends LedgerEntry {
    }

    // ── Fixture helpers ───────────────────────────────────────────────────────

    private TestLedgerEntry decision(final String actorId, final Instant occurredAt) {
        final TestLedgerEntry e = new TestLedgerEntry();
        e.id = UUID.randomUUID();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.HUMAN;
        e.occurredAt = occurredAt;
        return e;
    }

    /** Attestation with occurredAt = now (zero age, full recency weight = 1.0). */
    private LedgerAttestation attestation(final UUID entryId, final AttestationVerdict verdict) {
        return attestation(entryId, verdict, now);
    }

    /** Attestation with explicit occurredAt for recency-sensitive tests. */
    private LedgerAttestation attestation(final UUID entryId, final AttestationVerdict verdict,
            final Instant occurredAt) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.attestorId = "peer";
        a.attestorType = ActorType.HUMAN;
        a.verdict = verdict;
        a.confidence = 0.9;
        a.occurredAt = occurredAt;
        return a;
    }

    // ── Empty history ─────────────────────────────────────────────────────────

    @Test
    void emptyHistory_returnsNeutralScore() {
        final TrustScoreComputer.ActorScore score = computer.compute(List.of(), Map.of(), now);

        assertThat(score.trustScore()).isCloseTo(0.5, within(0.01));
        assertThat(score.decisionCount()).isEqualTo(0);
    }

    @Test
    void emptyHistory_priorAlphaBeta_one() {
        final TrustScoreComputer.ActorScore score = computer.compute(List.of(), Map.of(), now);

        assertThat(score.alpha()).isCloseTo(1.0, within(0.001));
        assertThat(score.beta()).isCloseTo(1.0, within(0.001));
    }

    @Test
    void emptyHistory_countersAreZero() {
        final TrustScoreComputer.ActorScore score = computer.compute(List.of(), Map.of(), now);

        assertThat(score.overturnedCount()).isEqualTo(0);
        assertThat(score.attestationPositive()).isEqualTo(0);
        assertThat(score.attestationNegative()).isEqualTo(0);
    }

    // ── Unattested decisions ──────────────────────────────────────────────────

    @Test
    void unattestedDecision_scoresNeutral() {
        // Unattested decisions contribute nothing. Prior Beta(1,1) → 0.5.
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));

        final TrustScoreComputer.ActorScore score = computer.compute(List.of(d), Map.of(), now);

        assertThat(score.trustScore()).isCloseTo(0.5, within(0.01));
        assertThat(score.decisionCount()).isEqualTo(1);
    }

    @Test
    void multipleUnattestedDecisions_stillNeutral() {
        final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.DAYS));
        final TestLedgerEntry d2 = decision("alice", now.minus(2, ChronoUnit.DAYS));
        final TestLedgerEntry d3 = decision("alice", now.minus(3, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d1, d2, d3), Map.of(), now);

        assertThat(score.trustScore()).isCloseTo(0.5, within(0.01));
        assertThat(score.decisionCount()).isEqualTo(3);
    }

    // ── Single positive attestation ───────────────────────────────────────────

    @Test
    void onePositiveAttestation_scoreAboveNeutralNotAtMax() {
        // α=1+1=2, β=1 → score = 2/3 ≈ 0.667. Not 1.0 — uncertainty acknowledged.
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation a = attestation(d.id, AttestationVerdict.SOUND);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);

        assertThat(score.trustScore()).isCloseTo(2.0 / 3.0, within(0.01));
        assertThat(score.alpha()).isCloseTo(2.0, within(0.01));
        assertThat(score.beta()).isCloseTo(1.0, within(0.01));
        assertThat(score.attestationPositive()).isEqualTo(1);
        assertThat(score.attestationNegative()).isEqualTo(0);
    }

    @Test
    void endorsedAttestation_countsAsPositive() {
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation a = attestation(d.id, AttestationVerdict.ENDORSED);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);

        assertThat(score.attestationPositive()).isEqualTo(1);
        assertThat(score.trustScore()).isGreaterThan(0.5);
    }

    // ── Single negative attestation ───────────────────────────────────────────

    @Test
    void oneNegativeAttestation_scoreBelowNeutralNotAtMin() {
        // α=1, β=1+1=2 → score = 1/3 ≈ 0.333. Not 0.0 — uncertainty acknowledged.
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation a = attestation(d.id, AttestationVerdict.FLAGGED);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);

        assertThat(score.trustScore()).isCloseTo(1.0 / 3.0, within(0.01));
        assertThat(score.alpha()).isCloseTo(1.0, within(0.01));
        assertThat(score.beta()).isCloseTo(2.0, within(0.01));
        assertThat(score.overturnedCount()).isEqualTo(1);
        assertThat(score.attestationNegative()).isEqualTo(1);
    }

    @Test
    void challengedAttestation_countsAsNegative() {
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation a = attestation(d.id, AttestationVerdict.CHALLENGED);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);

        assertThat(score.attestationNegative()).isEqualTo(1);
        assertThat(score.trustScore()).isLessThan(0.5);
    }

    // ── Uncertainty capture ───────────────────────────────────────────────────

    @Test
    void moreEvidenceYieldsHigherConfidence() {
        // Key Beta property: 1 positive → 0.667; 100 positives → 0.990.
        final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation onePositive = attestation(d1.id, AttestationVerdict.SOUND);

        final TestLedgerEntry d2 = decision("bob", now.minus(1, ChronoUnit.HOURS));
        final List<LedgerAttestation> manyPositive = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            manyPositive.add(attestation(d2.id, AttestationVerdict.SOUND));
        }

        final TrustScoreComputer.ActorScore scoreOne = computer.compute(
                List.of(d1), Map.of(d1.id, List.of(onePositive)), now);
        final TrustScoreComputer.ActorScore scoreMany = computer.compute(
                List.of(d2), Map.of(d2.id, manyPositive), now);

        assertThat(scoreMany.trustScore()).isGreaterThan(scoreOne.trustScore());
        assertThat(scoreMany.trustScore()).isCloseTo(101.0 / 102.0, within(0.01));
    }

    // ── Mixed attestations ────────────────────────────────────────────────────

    @Test
    void mixedAttestations_twoPositiveOneNegative_scoreAboveNeutral() {
        // α=1+2=3, β=1+1=2 → score = 3/5 = 0.6
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation s1 = attestation(d.id, AttestationVerdict.SOUND);
        final LedgerAttestation s2 = attestation(d.id, AttestationVerdict.SOUND);
        final LedgerAttestation f1 = attestation(d.id, AttestationVerdict.FLAGGED);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(s1, s2, f1)), now);

        assertThat(score.trustScore()).isCloseTo(0.6, within(0.01));
        assertThat(score.attestationPositive()).isEqualTo(2);
        assertThat(score.attestationNegative()).isEqualTo(1);
    }

    @Test
    void mixedAttestations_onePositiveTwoNegative_scoreBelowNeutral() {
        // α=1+1=2, β=1+2=3 → score = 2/5 = 0.4
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation sound = attestation(d.id, AttestationVerdict.SOUND);
        final LedgerAttestation f1 = attestation(d.id, AttestationVerdict.FLAGGED);
        final LedgerAttestation f2 = attestation(d.id, AttestationVerdict.FLAGGED);

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(sound, f1, f2)), now);

        assertThat(score.trustScore()).isCloseTo(0.4, within(0.01));
        assertThat(score.attestationPositive()).isEqualTo(1);
        assertThat(score.attestationNegative()).isEqualTo(2);
    }

    // ── Multiple decisions ────────────────────────────────────────────────────

    @Test
    void multipleDecisions_mixedHistory_correctCounters() {
        final TestLedgerEntry clean1 = decision("alice", now.minus(1, ChronoUnit.DAYS));
        final TestLedgerEntry clean2 = decision("alice", now.minus(2, ChronoUnit.DAYS));
        final TestLedgerEntry bad = decision("alice", now.minus(3, ChronoUnit.DAYS));
        final LedgerAttestation flagged = attestation(bad.id, AttestationVerdict.FLAGGED,
                now.minus(3, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(clean1, clean2, bad),
                Map.of(bad.id, List.of(flagged)),
                now);

        assertThat(score.decisionCount()).isEqualTo(3);
        assertThat(score.overturnedCount()).isEqualTo(1);
        assertThat(score.trustScore()).isGreaterThan(0.0);
        assertThat(score.trustScore()).isLessThan(0.5); // net negative
    }

    // ── Recency weighting ─────────────────────────────────────────────────────

    @Test
    void recentPositiveAttestation_outweighsOldNegative() {
        // Recent positive: recencyWeight ≈ 1.0 → α increases by ~1.0
        // Old negative (180 days, halfLife=90): recencyWeight = 2^(-2) = 0.25 → β increases by 0.25
        final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.DAYS));
        final LedgerAttestation recentSound = attestation(
                d1.id, AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS));

        final TestLedgerEntry d2 = decision("alice", now.minus(180, ChronoUnit.DAYS));
        final LedgerAttestation oldFlagged = attestation(
                d2.id, AttestationVerdict.FLAGGED, now.minus(180, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d1, d2),
                Map.of(d1.id, List.of(recentSound), d2.id, List.of(oldFlagged)),
                now);

        assertThat(score.trustScore()).isGreaterThan(0.5);
    }

    @Test
    void attestationAgeUsed_notDecisionAge() {
        // One old decision (365 days ago) with a recent attestation (1 day ago).
        // The attestation's own occurredAt is used for recency — weight ≈ 1.0, not ~0.
        final TestLedgerEntry old = decision("alice", now.minus(365, ChronoUnit.DAYS));
        final LedgerAttestation recentOnOldDecision = attestation(
                old.id, AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore usingAttestationAge = computer.compute(
                List.of(old), Map.of(old.id, List.of(recentOnOldDecision)), now);

        // If decision age were used (365 days, halfLife=90): weight = 2^(-4) ≈ 0.0625
        // → α ≈ 1.0625, score ≈ 0.516 (barely above neutral)
        // If attestation age used (1 day): weight ≈ 1.0
        // → α ≈ 2.0, score ≈ 0.667 (clearly above neutral)
        assertThat(usingAttestationAge.trustScore()).isGreaterThan(0.6);
    }

    @Test
    void shortHalfLife_oldAttestationHasMinimalWeight() {
        final TrustScoreComputer shortHalfLife = new TrustScoreComputer(30);

        final TestLedgerEntry d1 = decision("alice", now.minus(1, ChronoUnit.DAYS));
        final LedgerAttestation recentSound = attestation(
                d1.id, AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS));

        final TestLedgerEntry d2 = decision("alice", now.minus(365, ChronoUnit.DAYS));
        final LedgerAttestation oldFlagged = attestation(
                d2.id, AttestationVerdict.FLAGGED, now.minus(365, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore score = shortHalfLife.compute(
                List.of(d1, d2),
                Map.of(d1.id, List.of(recentSound), d2.id, List.of(oldFlagged)),
                now);

        assertThat(score.trustScore()).isGreaterThan(0.5);
    }

    // ── Score bounds ──────────────────────────────────────────────────────────

    @Test
    void score_alwaysWithinZeroToOne() {
        final TestLedgerEntry d = decision("alice", now.minus(1, ChronoUnit.DAYS));
        final List<LedgerAttestation> many = new ArrayList<>();
        for (int i = 0; i < 50; i++) {
            many.add(attestation(d.id, AttestationVerdict.FLAGGED));
        }

        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, many), now);

        assertThat(score.trustScore()).isGreaterThanOrEqualTo(0.0);
        assertThat(score.trustScore()).isLessThanOrEqualTo(1.0);
    }

    // ── Default half-life ─────────────────────────────────────────────────────

    @Test
    void zeroHalfLifeDays_defaultsTo90() {
        final TrustScoreComputer computer0 = new TrustScoreComputer(0);
        final TrustScoreComputer computer90 = new TrustScoreComputer(90);

        final TestLedgerEntry d = decision("alice", now.minus(30, ChronoUnit.DAYS));
        final LedgerAttestation a = attestation(
                d.id, AttestationVerdict.SOUND, now.minus(30, ChronoUnit.DAYS));

        final TrustScoreComputer.ActorScore s0 = computer0.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);
        final TrustScoreComputer.ActorScore s90 = computer90.compute(
                List.of(d), Map.of(d.id, List.of(a)), now);

        assertThat(s0.trustScore()).isCloseTo(s90.trustScore(), within(0.001));
    }
}
```

- [ ] **Step 3: Run tests — all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreComputerTest -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: `Tests run: 22, Failures: 0` and `BUILD SUCCESS`.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java
git commit -m "$(cat <<'EOF'
feat(trust): replace ForgivenessParams with Bayesian Beta accumulation

TrustScoreComputer now accumulates alpha/beta per attestation with
recency-decayed contributions. Score = alpha/(alpha+beta). Prior Beta(1,1)
gives 0.5 with no history. Unattested decisions contribute nothing.
ForgivenessParams and two-arg constructor removed.

Refs #28
EOF
)"
```

---

## Task 3: Update `ActorTrustScore` entity and V1001 schema

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`
- Modify: `runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql`

- [ ] **Step 1: Replace `ActorTrustScore.java`**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.Table;

/**
 * Bayesian Beta trust score for a decision-making actor.
 * Plain {@code @Entity} — queries via {@code @NamedQuery} + EntityManager.
 */
@Entity
@Table(name = "actor_trust_score")
@NamedQuery(name = "ActorTrustScore.findAll", query = "SELECT s FROM ActorTrustScore s")
public class ActorTrustScore {

    @Id
    @Column(name = "actor_id")
    public String actorId;

    @Enumerated(EnumType.STRING)
    @Column(name = "actor_type")
    public ActorType actorType;

    @Column(name = "trust_score")
    public double trustScore;

    @Column(name = "alpha_value")
    public double alpha;

    @Column(name = "beta_value")
    public double beta;

    @Column(name = "decision_count")
    public int decisionCount;

    @Column(name = "overturned_count")
    public int overturnedCount;

    @Column(name = "attestation_positive")
    public int attestationPositive;

    @Column(name = "attestation_negative")
    public int attestationNegative;

    @Column(name = "last_computed_at")
    public Instant lastComputedAt;
}
```

- [ ] **Step 2: Replace `V1001__actor_trust_score.sql`**

```sql
-- CaseHub Ledger — Bayesian Beta actor trust score table (V1001)
-- Compatible with H2 (dev/test) and PostgreSQL (production)
--
-- actor_trust_score: nightly-computed Bayesian Beta trust scores per actor.
-- actor_id is the primary key — one row per actor, upserted on each nightly run.
-- alpha_value and beta_value are the Beta distribution parameters.
-- Score = alpha_value / (alpha_value + beta_value).

CREATE TABLE actor_trust_score (
    actor_id             VARCHAR(255)    NOT NULL,
    actor_type           VARCHAR(20),
    trust_score          DOUBLE          NOT NULL,
    alpha_value          DOUBLE          NOT NULL,
    beta_value           DOUBLE          NOT NULL,
    decision_count       INT             NOT NULL,
    overturned_count     INT             NOT NULL,
    attestation_positive INT             NOT NULL,
    attestation_negative INT             NOT NULL,
    last_computed_at     TIMESTAMP,
    CONSTRAINT pk_actor_trust_score PRIMARY KEY (actor_id)
);
```

- [ ] **Step 3: Compile check (tests will fail — repository not yet updated)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | grep -E "ERROR|BUILD"
```

Expected: `BUILD SUCCESS` (compile only — test compilation may fail until Task 4).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java \
        runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql
git commit -m "$(cat <<'EOF'
feat(schema): replace appeal_count with alpha_value/beta_value in actor_trust_score

Drops the always-zero appeal_count column. Adds alpha_value and beta_value
(DOUBLE) to store the Bayesian Beta posterior parameters computed by
TrustScoreComputer.

Refs #28
EOF
)"
```

---

## Task 4: Update `ActorTrustScoreRepository` SPI and JPA implementation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`

- [ ] **Step 1: Replace `ActorTrustScoreRepository.java`**

```java
package io.casehub.ledger.runtime.repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;

/** SPI for persisting and querying {@link ActorTrustScore} records. */
public interface ActorTrustScoreRepository {

    /**
     * Find the trust score for an actor, or empty if none computed yet.
     *
     * @param actorId the actor's identity string
     * @return the computed score, or empty if not yet computed
     */
    Optional<ActorTrustScore> findByActorId(String actorId);

    /**
     * Upsert (insert or update) a trust score for the given actor.
     *
     * @param actorId the actor's identity string
     * @param actorType the type of actor
     * @param trustScore the computed trust score in [0.0, 1.0]
     * @param decisionCount total number of EVENT entries attributed to this actor
     * @param overturnedCount number of decisions with at least one negative attestation
     * @param alpha final α value from the Beta distribution
     * @param beta final β value from the Beta distribution
     * @param attestationPositive total positive attestation count
     * @param attestationNegative total negative attestation count
     * @param lastComputedAt the timestamp of this computation
     */
    void upsert(String actorId, ActorType actorType, double trustScore,
            int decisionCount, int overturnedCount, double alpha, double beta,
            int attestationPositive, int attestationNegative,
            Instant lastComputedAt);

    /**
     * Return all computed trust scores.
     *
     * @return list of all actor trust scores; empty if none computed yet
     */
    List<ActorTrustScore> findAll();
}
```

- [ ] **Step 2: Replace `JpaActorTrustScoreRepository.java`**

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;

/**
 * JPA / EntityManager implementation of {@link ActorTrustScoreRepository}.
 *
 * <p>
 * Upsert is implemented as a find-then-update to remain compatible with both H2 (dev/test)
 * and PostgreSQL (production) without requiring database-specific SQL syntax.
 */
@ApplicationScoped
public class JpaActorTrustScoreRepository implements ActorTrustScoreRepository {

    @Inject
    EntityManager em;

    /** {@inheritDoc} */
    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return Optional.ofNullable(em.find(ActorTrustScore.class, actorId));
    }

    /** {@inheritDoc} */
    @Override
    @Transactional
    public void upsert(final String actorId, final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount,
            final double alpha, final double beta,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {

        ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        if (score == null) {
            score = new ActorTrustScore();
            score.actorId = actorId;
        }
        score.actorType = actorType;
        score.trustScore = trustScore;
        score.alpha = alpha;
        score.beta = beta;
        score.decisionCount = decisionCount;
        score.overturnedCount = overturnedCount;
        score.attestationPositive = attestationPositive;
        score.attestationNegative = attestationNegative;
        score.lastComputedAt = lastComputedAt;
        em.merge(score);
    }

    /** {@inheritDoc} */
    @Override
    public List<ActorTrustScore> findAll() {
        return em.createNamedQuery("ActorTrustScore.findAll", ActorTrustScore.class)
                .getResultList();
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java
git commit -m "$(cat <<'EOF'
feat(repository): update ActorTrustScoreRepository upsert — alpha/beta replace appealCount

Refs #28
EOF
)"
```

---

## Task 5: Update `TrustScoreJob` and `LedgerConfig`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Step 1: Replace `TrustScoreJob.java`**

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.scheduler.Scheduled;

/**
 * Nightly scheduled job that recomputes Bayesian Beta trust scores for all
 * decision-making actors in the ledger.
 *
 * <p>
 * The job is gated by {@code quarkus.ledger.trust-score.enabled}. When disabled, the
 * scheduled trigger fires but immediately returns without doing any work.
 *
 * <p>
 * {@link #runComputation()} is exposed with package-accessible visibility for direct
 * invocation in integration tests where the scheduler is disabled via a test profile.
 */
@ApplicationScoped
public class TrustScoreJob {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    LedgerConfig config;

    @Scheduled(every = "24h", identity = "ledger-trust-score-job")
    @Transactional
    public void computeTrustScores() {
        if (!config.trustScore().enabled()) {
            return;
        }
        runComputation();
    }

    @Transactional
    public void runComputation() {
        final TrustScoreComputer computer = new TrustScoreComputer(
                config.trustScore().decayHalfLifeDays());
        final Instant now = Instant.now();

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
    }
}
```

- [ ] **Step 2: Remove `ForgivenessConfig` from `LedgerConfig.java`**

Remove the `forgiveness()` method and the entire `ForgivenessConfig` inner interface from `TrustScoreConfig`. The `TrustScoreConfig` interface after the edit:

```java
/** Bayesian Beta reputation computation settings. */
interface TrustScoreConfig {

    /**
     * When {@code true}, a nightly scheduled job computes Bayesian Beta trust scores
     * from ledger history. Off by default — trust scores require accumulated history to be
     * meaningful; enabling on a new deployment produces unreliable early scores.
     *
     * @return {@code true} if trust score computation is enabled; {@code false} by default
     */
    @WithDefault("false")
    boolean enabled();

    /**
     * Exponential decay half-life in days for attestation recency weighting.
     * Attestations older than this are down-weighted relative to recent ones.
     *
     * @return half-life in days (default 90)
     */
    @WithDefault("90")
    int decayHalfLifeDays();

    /**
     * When {@code true}, trust scores influence routing suggestions via CDI events.
     * Scores must be enabled and accumulated before enabling routing.
     *
     * @return {@code true} if trust-score-based routing is enabled; {@code false} by default
     */
    @WithDefault("false")
    boolean routingEnabled();
}
```

In `LedgerConfig.java`, find the existing `TrustScoreConfig` interface (lines ~208–285) and replace the entire body with the above, removing `forgiveness()` and `ForgivenessConfig`.

- [ ] **Step 3: Run full test suite — should be close to all passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD|FAIL|ERROR" | tail -10
```

Expected: `BUILD SUCCESS`. If `TrustScoreForgivenessIT` fails, that's expected — it is removed in Task 6.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
        runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git commit -m "$(cat <<'EOF'
feat(job,config): remove ForgivenessParams wiring from TrustScoreJob and LedgerConfig

TrustScoreJob uses single-arg TrustScoreComputer constructor. ForgivenessConfig
sub-interface and forgiveness() accessor removed from LedgerConfig.TrustScoreConfig.

Refs #28
EOF
)"
```

---

## Task 6: Replace `TrustScoreForgivenessIT` with `TrustScoreIT`

**Files:**
- Delete: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreIT.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Delete `TrustScoreForgivenessIT.java`**

```bash
rm runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java
```

- [ ] **Step 2: Remove forgiveness profile, add trust-score-test profile in `application.properties`**

Find the forgiveness block in `application.properties`:
```properties
# Trust score + forgiveness profile (used by TrustScoreForgivenessIT)
%forgiveness-test.quarkus.ledger.trust-score.enabled=true
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.enabled=true
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.frequency-threshold=3
%forgiveness-test.quarkus.ledger.trust-score.forgiveness.half-life-days=30
```

Replace with:
```properties
# Trust score profile (used by TrustScoreIT)
%trust-score-test.quarkus.ledger.trust-score.enabled=true
```

- [ ] **Step 3: Create `TrustScoreIT.java`**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
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

/**
 * End-to-end integration tests for the Bayesian Beta trust scoring pipeline.
 *
 * <p>
 * Runs the full {@link TrustScoreJob} against an H2 database populated with ledger
 * entries and attestations. Verifies score values, alpha/beta parameters, and
 * recency-weighting behaviour.
 */
@QuarkusTest
@TestProfile(TrustScoreIT.TrustScoreTestProfile.class)
class TrustScoreIT {

    public static class TrustScoreTestProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "trust-score-test";
        }
    }

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    // ── Happy path: no history → neutral score ────────────────────────────────

    @Test
    @Transactional
    void noAttestations_neutralScore() {
        final String actorId = "agent-no-att-" + UUID.randomUUID();

        seedDecision(actorId, Instant.now().minus(1, ChronoUnit.DAYS), null);
        seedDecision(actorId, Instant.now().minus(2, ChronoUnit.DAYS), null);

        trustScoreJob.runComputation();

        final ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isCloseTo(0.5, within(0.01));
        assertThat(score.alpha).isCloseTo(1.0, within(0.01));
        assertThat(score.beta).isCloseTo(1.0, within(0.01));
        assertThat(score.decisionCount).isEqualTo(2);
    }

    // ── Happy path: positive attestations → high score ────────────────────────

    @Test
    @Transactional
    void allPositiveAttestations_highScore() {
        final String actorId = "agent-positive-" + UUID.randomUUID();
        final Instant recentTime = Instant.now().minus(1, ChronoUnit.DAYS);

        seedDecision(actorId, recentTime, AttestationVerdict.SOUND);
        seedDecision(actorId, recentTime.minus(1, ChronoUnit.DAYS), AttestationVerdict.ENDORSED);
        seedDecision(actorId, recentTime.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND);

        trustScoreJob.runComputation();

        final ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isGreaterThan(0.75);
        assertThat(score.alpha).isGreaterThan(score.beta);
        assertThat(score.attestationPositive).isEqualTo(3);
        assertThat(score.attestationNegative).isEqualTo(0);
    }

    // ── Happy path: negative attestations → low score ─────────────────────────

    @Test
    @Transactional
    void allNegativeAttestations_lowScore() {
        final String actorId = "agent-negative-" + UUID.randomUUID();
        final Instant recentTime = Instant.now().minus(1, ChronoUnit.DAYS);

        seedDecision(actorId, recentTime, AttestationVerdict.FLAGGED);
        seedDecision(actorId, recentTime.minus(1, ChronoUnit.DAYS), AttestationVerdict.CHALLENGED);

        trustScoreJob.runComputation();

        final ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isLessThan(0.4);
        assertThat(score.beta).isGreaterThan(score.alpha);
        assertThat(score.overturnedCount).isEqualTo(2);
    }

    // ── Correctness: alpha/beta values match expected Beta posterior ──────────

    @Test
    @Transactional
    void alphaBeta_matchExpectedPosterior() {
        // 1 SOUND attestation, effectively age=0 → recencyWeight ≈ 1.0
        // α = 1.0 (prior) + 1.0 = 2.0, β = 1.0 (prior), score = 2/3 ≈ 0.667
        final String actorId = "agent-posterior-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDecisionWithAttestationAt(actorId, now.minus(1, ChronoUnit.HOURS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.HOURS));

        trustScoreJob.runComputation();

        final ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isCloseTo(2.0 / 3.0, within(0.02));
        assertThat(score.alpha).isCloseTo(2.0, within(0.05));
        assertThat(score.beta).isCloseTo(1.0, within(0.05));
    }

    // ── End-to-end: recency — old negative, recent positive → score > 0.5 ────

    @Test
    @Transactional
    void oldNegative_recentPositive_scoreAboveNeutral() {
        final String actorId = "agent-recency-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // Old failure 180 days ago: recencyWeight = 2^(-180/90) = 0.25 → β += 0.25
        seedDecisionWithAttestationAt(actorId, now.minus(180, ChronoUnit.DAYS),
                AttestationVerdict.FLAGGED, now.minus(180, ChronoUnit.DAYS));

        // Recent endorsement 1 day ago: recencyWeight ≈ 1.0 → α += ~1.0
        seedDecisionWithAttestationAt(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS));

        trustScoreJob.runComputation();

        final ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        assertThat(score).isNotNull();
        // α ≈ 2.0, β ≈ 1.25 → score ≈ 2.0/3.25 ≈ 0.615
        assertThat(score.trustScore).isGreaterThan(0.5);
    }

    // ── Fixtures ──────────────────────────────────────────────────────────────

    /** Seeds a decision with an attestation whose occurredAt = decisionTime + 60s. */
    private void seedDecision(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdictOrNull) {
        seedDecisionWithAttestationAt(actorId, decisionTime, verdictOrNull,
                verdictOrNull != null ? decisionTime.plusSeconds(60) : null);
    }

    private void seedDecisionWithAttestationAt(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdictOrNull, final Instant attestationTime) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = decisionTime;
        repo.save(entry);

        if (verdictOrNull != null) {
            final LedgerAttestation att = new LedgerAttestation();
            att.id = UUID.randomUUID();
            att.ledgerEntryId = entry.id;
            att.subjectId = entry.subjectId;
            att.attestorId = "compliance-bot";
            att.attestorType = ActorType.AGENT;
            att.verdict = verdictOrNull;
            att.confidence = 0.9;
            att.occurredAt = attestationTime;
            em.persist(att);
        }
    }
}
```

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: all tests pass, `BUILD SUCCESS`. Total count will be slightly different from 131 (forgiveness IT removed, Beta IT added).

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/TrustScoreIT.java \
        runtime/src/test/resources/application.properties
git rm runtime/src/test/java/io/casehub/ledger/service/TrustScoreForgivenessIT.java
git commit -m "$(cat <<'EOF'
test(trust): replace TrustScoreForgivenessIT with TrustScoreIT — Beta integration tests

5 tests: no history neutral, all positive, all negative, alpha/beta posterior
correctness, and recency weighting end-to-end.

Refs #28
EOF
)"
```

---

## Task 7: Update docs — DESIGN.md, RESEARCH.md, ADRs

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `docs/RESEARCH.md`
- Modify: `adr/0001-forgiveness-mechanism-omits-severity-dimension.md`
- Create: `adr/0003-bayesian-beta-trust-model.md`
- Modify: `adr/INDEX.md`

- [ ] **Step 1: Update trust scoring section in `DESIGN.md`**

Find the **Forgiveness sub-config** table (lines around 228–238) and replace the forgiveness block with:

```markdown
| `trust-score.enabled` | `false` | Enable nightly Bayesian Beta trust score computation (requires historical data) |
| `trust-score.decay-half-life-days` | `90` | Recency decay half-life for attestation age weighting |
| `trust-score.routing-enabled` | `false` | Influence routing via CDI events based on trust scores |
```

Then find the trust scoring algorithm description and update it to describe the Beta model. Find the line mentioning "EigenTrust-inspired" and replace the algorithm description with:

```markdown
**Trust scoring** uses a Bayesian Beta model. For each actor, all attestations across all
decisions accumulate into a Beta distribution: `α` for positive verdicts (SOUND, ENDORSED),
`β` for negative verdicts (FLAGGED, CHALLENGED). Each contribution is recency-weighted:
`weight = 2^(-ageInDays / decayHalfLifeDays)` using the attestation's own timestamp.
Prior is Beta(1,1) → score 0.5 with no history. Score = α/(α+β).

`ActorTrustScore` stores `trust_score`, `alpha_value`, `beta_value`, and diagnostic
counters. `TrustScoreJob` runs nightly when enabled.
```

Find the **Bayesian trust weighting** entry in the roadmap/tracker section and mark it done:
```markdown
| **Bayesian trust weighting** | ✅ Done | Bayesian Beta model: per-attestation recency weighting, alpha/beta posterior, ForgivenessParams removed. See ADR 0003. |
```

Find the **Forgiveness mechanism** progress tracker entry and update to:
```markdown
| **Forgiveness mechanism** | ✅ Superseded | Replaced by Bayesian Beta model (ADR 0003). ForgivenessParams removed. |
```

- [ ] **Step 2: Mark priority #6 done in `RESEARCH.md`**

Find the priority #6 row:
```markdown
| 6 | **Time-weighted Bayesian trust** | M | Medium | ★★★ | Current exponential decay...
```

Replace with:
```markdown
| 6 | **Time-weighted Bayesian trust** | ~~M~~ | ~~Medium~~ | ✅ Done | Bayesian Beta model: per-attestation α/β accumulation with recency weighting. `ForgivenessParams` removed. See ADR 0003. |
```

- [ ] **Step 3: Mark ADR 0001 superseded**

At the top of `adr/0001-forgiveness-mechanism-omits-severity-dimension.md`, change:
```markdown
Status: Accepted
```
to:
```markdown
Status: Superseded by ADR 0003
```

- [ ] **Step 4: Create `adr/0003-bayesian-beta-trust-model.md`**

```markdown
# 0003 — Bayesian Beta model replaces ForgivenessParams

Date: 2026-04-20
Status: Accepted

## Context and Problem Statement

The `TrustScoreComputer` classified each decision as 1.0 / 0.5 / 0.0 and computed a
recency-weighted average. Two problems: (1) no uncertainty model — an actor with 1 positive
attestation scored identically to one with 100 positives; (2) `ForgivenessParams` was a
patch on top of a coarse model, adding complexity without fixing the underlying issue.

## Decision

Replace the classification + weighted-average approach with a Bayesian Beta distribution.

Each attestation contributes a recency-weighted increment to α (positive) or β (negative)
using the attestation's own timestamp: `weight = 2^(-ageInDays / halfLifeDays)`. Prior is
Beta(1,1). Score = α/(α+β). Unattested decisions contribute nothing — the prior handles
"no information" with score 0.5.

`ForgivenessParams` is removed entirely. The Beta model supersedes both of its concerns:
recency of negative attestations fades naturally via weighting on β; frequency effects
are implicit in the α/β ratio.

## Consequences

* **Better:** uncertainty is modelled — evidence quality is captured, not just direction.
* **Better:** old negative attestations fade without a separate forgiveness mechanism.
* **Better:** simpler API — one constructor parameter, no `ForgivenessParams`.
* **Changed:** unattested decisions no longer score 1.0 (clean) — they score 0.5 (unknown).
  Consumers relying on "clean by default" behaviour must add attestations to signal quality.
* **Supersedes:** ADR 0001 (forgiveness severity dimension — forgiveness mechanism removed).
```

- [ ] **Step 5: Update `adr/INDEX.md`**

Add entry for ADR 0003 and update ADR 0001 status:

```markdown
| 0001 | Forgiveness mechanism omits severity | Superseded by 0003 |
| 0003 | Bayesian Beta model replaces ForgivenessParams | Accepted |
```

- [ ] **Step 6: Run full test suite — final green check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 7: Commit**

```bash
git add docs/DESIGN.md docs/RESEARCH.md \
        adr/0001-forgiveness-mechanism-omits-severity-dimension.md \
        adr/0003-bayesian-beta-trust-model.md \
        adr/INDEX.md
git commit -m "$(cat <<'EOF'
docs: update DESIGN.md, RESEARCH.md, ADRs for Bayesian Beta trust model

Marks research priority #6 done. ADR 0001 superseded by ADR 0003.
DESIGN.md trust scoring section updated to describe Beta model.
Forgiveness config table removed.

Closes #28
EOF
)"
```

---

## Verification

After all tasks complete:

```bash
# All tests pass
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:"

# No ForgivenessParams remaining anywhere
grep -r "ForgivenessParams\|forgiveness\|appealCount\|appeal_count" \
  runtime/src/main runtime/src/test --include="*.java" --include="*.sql" --include="*.properties"

# All commits reference #28
git log --oneline -10 | grep "#28"
```
