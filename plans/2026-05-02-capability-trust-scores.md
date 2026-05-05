# Capability-Scoped Trust Scores Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `GlobalScoreStrategy` SPI with three implementations, extend `TrustScoreJob` to compute per-capability Beta scores, and wire up `TrustGateService.meetsThreshold(actorId, capabilityTag, minTrust)` with capability-then-global fallback.

**Architecture:** `GlobalScoreStrategy` follows the `DecayFunction` CDI pattern — a `@FunctionalInterface`-style SPI with `@DefaultBean` for Option B (all attestations). `TrustScoreJob` computes capability scores first (needed for Option C's `derive()`), then global. No schema changes — `actor_trust_score` already has `score_type`/`scope_key` columns from V1001.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI `@DefaultBean`/`@Alternative`, Hibernate ORM, JUnit 5, AssertJ, H2 (tests).

**Issues:** Closes #61 · Part of epic #50

---

## Pre-flight

Before every task, verify both IntelliJ MCPs are available:
- `mcp__intellij__get_project_modules` with `projectPath=/Users/mdproctor/claude/casehub/ledger`
- `mcp__intellij-index__ide_index_status` with `project_path=/Users/mdproctor/claude/casehub/ledger`

If either fails, **stop and report to the user.** Do not proceed.

**Before using any bash tool**, check if the operation can be performed via IntelliJ MCP first (build, file problems, symbol search, rename refactoring).

---

## File Map

| Action | File | What changes |
|---|---|---|
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/GlobalScoreStrategy.java` | New SPI interface |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/AllAttestationsGlobalStrategy.java` | Option B — `@DefaultBean` |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/ExplicitGlobalAttestationsStrategy.java` | Option A — `@Alternative` |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/FrequencyWeightedGlobalStrategy.java` | Option C — `@Alternative` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` | Inject strategy; capability pass before global |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java` | Wire capability overload; add `currentScore(id, tag)` |
| Create | `runtime/src/test/java/io/casehub/ledger/service/GlobalScoreStrategyTest.java` | Unit tests for all 3 strategies |
| Create | `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityIT.java` | IT tests for job + gate |
| Modify | `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java` | Add Phase 2 unit tests |
| Create | `adr/0008-global-score-strategy-spi.md` | ADR for the SPI design |
| Modify | `docs/DESIGN.md` | Implementation Tracker row for #61 |
| Modify | `docs/DESIGN-capabilities.md` | Update trust scoring section |
| Modify | `CLAUDE.md` | Add new strategy classes to project structure |

---

### Task 1: `GlobalScoreStrategy` SPI interface

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/GlobalScoreStrategy.java`

- [ ] **Step 1: Create the SPI interface**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import io.casehub.ledger.runtime.model.LedgerAttestation;

/**
 * SPI for determining which attestations contribute to the GLOBAL {@link io.casehub.ledger.runtime.model.ActorTrustScore}.
 *
 * <p>
 * Three built-in implementations reflect the three literature-backed positions on what
 * the global score should aggregate:
 * <ul>
 * <li>{@link AllAttestationsGlobalStrategy} (default) — all attestations; Wang &amp; Vassileva root-node model</li>
 * <li>{@link ExplicitGlobalAttestationsStrategy} — only {@code capabilityTag = "*"} attestations</li>
 * <li>{@link FrequencyWeightedGlobalStrategy} — derived as frequency-weighted combination of capability scores; Fan et al. (2015)</li>
 * </ul>
 *
 * <p>
 * Alternative implementations register as {@code @Alternative} CDI beans and are activated via
 * {@code quarkus.arc.selected-alternatives=<fully-qualified-class-name>} in {@code application.properties}.
 *
 * <p>
 * See ADR 0008 and issue #61 for the research background and decision rationale.
 */
public interface GlobalScoreStrategy {

    /**
     * Select which attestations feed the global Beta model.
     *
     * @param all all attestations for the actor's decisions (may be empty)
     * @return the subset to include in the global Beta computation; return all, a filter, or empty
     */
    List<LedgerAttestation> selectAttestations(List<LedgerAttestation> all);

    /**
     * Optionally derive the global score from already-computed capability scores,
     * overriding the Beta model result from {@link #selectAttestations}.
     *
     * <p>
     * Called after all CAPABILITY rows are upserted. Return empty to keep the
     * Beta model score unchanged (default behaviour for Options A and B).
     * Option C returns a frequency-weighted combination here.
     *
     * @param capabilityScores map of capabilityTag to computed ActorScore for that capability
     * @param allAttestations all attestations for the actor (same list passed to {@link #selectAttestations})
     * @return derived score to override the Beta model result, or empty to use the Beta score
     */
    default Optional<TrustScoreComputer.ActorScore> derive(
            final Map<String, TrustScoreComputer.ActorScore> capabilityScores,
            final List<LedgerAttestation> allAttestations) {
        return Optional.empty();
    }
}
```

- [ ] **Step 2: Build to verify compilation**

Use IntelliJ: `mcp__intellij__build_project` with `projectPath=/Users/mdproctor/claude/casehub/ledger`, `filesToRebuild=["runtime/src/main/java/io/casehub/ledger/runtime/service/GlobalScoreStrategy.java"]`.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/GlobalScoreStrategy.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat: add GlobalScoreStrategy SPI for pluggable global trust score aggregation

Refs #61 · Part of epic #50"
```

---

### Task 2: Unit tests for all three strategy implementations (TDD — write failing tests first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/GlobalScoreStrategyTest.java`

- [ ] **Step 1: Write the failing test class**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.service.AllAttestationsGlobalStrategy;
import io.casehub.ledger.runtime.service.ExplicitGlobalAttestationsStrategy;
import io.casehub.ledger.runtime.service.FrequencyWeightedGlobalStrategy;
import io.casehub.ledger.runtime.service.TrustScoreComputer;

/**
 * Pure unit tests for all three {@link io.casehub.ledger.runtime.service.GlobalScoreStrategy} implementations.
 * No Quarkus runtime, no CDI.
 */
class GlobalScoreStrategyTest {

    // ── AllAttestationsGlobalStrategy ─────────────────────────────────────────

    @Test
    void allAttestations_returnsFullList() {
        final var strategy = new AllAttestationsGlobalStrategy();
        final List<LedgerAttestation> all = List.of(attestation("*"), attestation("security-review"));

        assertThat(strategy.selectAttestations(all)).isSameAs(all);
    }

    @Test
    void allAttestations_derive_returnsEmpty() {
        final var strategy = new AllAttestationsGlobalStrategy();
        final Map<String, TrustScoreComputer.ActorScore> caps = Map.of(
                "security-review", score(0.8));

        assertThat(strategy.derive(caps, List.of())).isEmpty();
    }

    @Test
    void allAttestations_emptyInput_returnsEmpty() {
        final var strategy = new AllAttestationsGlobalStrategy();

        assertThat(strategy.selectAttestations(List.of())).isEmpty();
    }

    // ── ExplicitGlobalAttestationsStrategy ───────────────────────────────────

    @Test
    void explicitGlobal_filtersToStarOnly() {
        final var strategy = new ExplicitGlobalAttestationsStrategy();
        final LedgerAttestation global = attestation(CapabilityTag.GLOBAL);
        final LedgerAttestation capability = attestation("security-review");

        final List<LedgerAttestation> result = strategy.selectAttestations(List.of(global, capability));

        assertThat(result).containsExactly(global);
    }

    @Test
    void explicitGlobal_emptyWhenNoGlobalAttestations() {
        final var strategy = new ExplicitGlobalAttestationsStrategy();
        final List<LedgerAttestation> capOnly = List.of(
                attestation("security-review"),
                attestation("style-review"));

        assertThat(strategy.selectAttestations(capOnly)).isEmpty();
    }

    @Test
    void explicitGlobal_allReturnedWhenAllAreGlobal() {
        final var strategy = new ExplicitGlobalAttestationsStrategy();
        final List<LedgerAttestation> all = List.of(attestation("*"), attestation("*"));

        assertThat(strategy.selectAttestations(all)).hasSize(2);
    }

    @Test
    void explicitGlobal_derive_returnsEmpty() {
        final var strategy = new ExplicitGlobalAttestationsStrategy();

        assertThat(strategy.derive(Map.of("security-review", score(0.9)), List.of())).isEmpty();
    }

    // ── FrequencyWeightedGlobalStrategy ──────────────────────────────────────

    @Test
    void frequencyWeighted_selectAttestations_alwaysEmpty() {
        final var strategy = new FrequencyWeightedGlobalStrategy();
        final List<LedgerAttestation> all = List.of(attestation("*"), attestation("security-review"));

        assertThat(strategy.selectAttestations(all)).isEmpty();
    }

    @Test
    void frequencyWeighted_derive_returnsEmptyWhenNoCapabilities() {
        final var strategy = new FrequencyWeightedGlobalStrategy();

        assertThat(strategy.derive(Map.of(), List.of())).isEmpty();
    }

    @Test
    void frequencyWeighted_derive_equalWeightsWhenEqualCounts() {
        // 1 attestation each for security-review (score 0.9) and style-review (score 0.5)
        // expected weighted score = (1/2) * 0.9 + (1/2) * 0.5 = 0.7
        final var strategy = new FrequencyWeightedGlobalStrategy();
        final List<LedgerAttestation> all = List.of(
                attestation("security-review"),
                attestation("style-review"));
        final Map<String, TrustScoreComputer.ActorScore> caps = Map.of(
                "security-review", score(0.9),
                "style-review", score(0.5));

        final var result = strategy.derive(caps, all);

        assertThat(result).isPresent();
        assertThat(result.get().trustScore()).isCloseTo(0.7, within(0.01));
    }

    @Test
    void frequencyWeighted_derive_higherWeightForMoreAttestedCapability() {
        // 3 security-review (score 0.8), 1 style-review (score 0.2)
        // expected = (3/4) * 0.8 + (1/4) * 0.2 = 0.6 + 0.05 = 0.65
        final var strategy = new FrequencyWeightedGlobalStrategy();
        final List<LedgerAttestation> all = List.of(
                attestation("security-review"),
                attestation("security-review"),
                attestation("security-review"),
                attestation("style-review"));
        final Map<String, TrustScoreComputer.ActorScore> caps = Map.of(
                "security-review", score(0.8),
                "style-review", score(0.2));

        final var result = strategy.derive(caps, all);

        assertThat(result).isPresent();
        assertThat(result.get().trustScore()).isCloseTo(0.65, within(0.01));
    }

    @Test
    void frequencyWeighted_derive_ignoresGlobalAttestationsInWeighting() {
        // Global ("*") attestations should NOT count toward capability weights
        final var strategy = new FrequencyWeightedGlobalStrategy();
        final List<LedgerAttestation> all = List.of(
                attestation(CapabilityTag.GLOBAL),  // should be ignored in weighting
                attestation(CapabilityTag.GLOBAL),  // should be ignored in weighting
                attestation("security-review"));     // weight = 1/1 = 1.0
        final Map<String, TrustScoreComputer.ActorScore> caps = Map.of(
                "security-review", score(0.8));

        final var result = strategy.derive(caps, all);

        assertThat(result).isPresent();
        assertThat(result.get().trustScore()).isCloseTo(0.8, within(0.01));
    }

    // ── Fixtures ─────────────────────────────────────────────────────────────

    private static LedgerAttestation attestation(final String capabilityTag) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = UUID.randomUUID();
        a.subjectId = UUID.randomUUID();
        a.attestorId = "peer";
        a.attestorType = ActorType.AGENT;
        a.verdict = AttestationVerdict.SOUND;
        a.confidence = 1.0;
        a.capabilityTag = capabilityTag;
        a.occurredAt = Instant.now();
        return a;
    }

    private static TrustScoreComputer.ActorScore score(final double trustScore) {
        return new TrustScoreComputer.ActorScore(trustScore, 2.0, 1.0, 1, 0, 1, 0);
    }
}
```

- [ ] **Step 2: Run to verify it fails (classes don't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=GlobalScoreStrategyTest -q 2>&1 | tail -10
```
Expected: FAIL — compilation error, `AllAttestationsGlobalStrategy` etc. not found.

---

### Task 3: `AllAttestationsGlobalStrategy` (Option B — default)

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AllAttestationsGlobalStrategy.java`

- [ ] **Step 1: Create the implementation**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkus.arc.DefaultBean;

import io.casehub.ledger.runtime.model.LedgerAttestation;

/**
 * Default {@link GlobalScoreStrategy}: all attestations feed the global Beta model.
 *
 * <p>
 * Backed by Wang &amp; Vassileva (2003) — the global trust score is the root node of the
 * Bayesian trust network, computed from all interactions; capability-specific scores are
 * derived views on top of it.
 *
 * <p>
 * This is the {@code @DefaultBean} — activated automatically when no alternative
 * {@link GlobalScoreStrategy} is selected.
 */
@ApplicationScoped
@DefaultBean
public class AllAttestationsGlobalStrategy implements GlobalScoreStrategy {

    @Override
    public List<LedgerAttestation> selectAttestations(final List<LedgerAttestation> all) {
        return all;
    }
}
```

- [ ] **Step 2: Run the AllAttestations tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest="GlobalScoreStrategyTest#allAttestations*" -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS, 3 tests PASSED.

---

### Task 4: `ExplicitGlobalAttestationsStrategy` (Option A)

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/ExplicitGlobalAttestationsStrategy.java`

- [ ] **Step 1: Create the implementation**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.LedgerAttestation;

/**
 * Alternative {@link GlobalScoreStrategy}: only attestations explicitly tagged as global
 * ({@code capabilityTag = }{@link CapabilityTag#GLOBAL}) feed the global Beta model.
 *
 * <p>
 * Semantic: a global attestation is an explicit statement that the verdict applies across
 * all capabilities. Capability-specific attestations do not influence the global score.
 *
 * <p>
 * Activate via {@code quarkus.arc.selected-alternatives=
 * io.casehub.ledger.runtime.service.ExplicitGlobalAttestationsStrategy}.
 */
@ApplicationScoped
@Alternative
public class ExplicitGlobalAttestationsStrategy implements GlobalScoreStrategy {

    @Override
    public List<LedgerAttestation> selectAttestations(final List<LedgerAttestation> all) {
        return all.stream()
                .filter(a -> CapabilityTag.GLOBAL.equals(a.capabilityTag))
                .collect(Collectors.toList());
    }
}
```

- [ ] **Step 2: Run the ExplicitGlobal tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest="GlobalScoreStrategyTest#explicitGlobal*" -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS, 4 tests PASSED.

---

### Task 5: `FrequencyWeightedGlobalStrategy` (Option C)

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/FrequencyWeightedGlobalStrategy.java`

- [ ] **Step 1: Create the implementation**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.LedgerAttestation;

/**
 * Alternative {@link GlobalScoreStrategy}: derives the global score as a frequency-weighted
 * combination of per-capability scores.
 *
 * <p>
 * Backed by Fan et al. (2015) — the overall trust score is computed as a weighted average of
 * dimension-specific scores, where each dimension's weight reflects its relative frequency:
 * {@code globalScore = Σ (count_i / totalCount) × trustScore_i}.
 *
 * <p>
 * {@link #selectAttestations} returns an empty list — the Beta model is bypassed.
 * {@link #derive} computes the weighted combination after capability scores are available.
 * Returns empty if no capability scores exist (no override; job uses Beta prior 0.5).
 *
 * <p>
 * Activate via {@code quarkus.arc.selected-alternatives=
 * io.casehub.ledger.runtime.service.FrequencyWeightedGlobalStrategy}.
 */
@ApplicationScoped
@Alternative
public class FrequencyWeightedGlobalStrategy implements GlobalScoreStrategy {

    @Override
    public List<LedgerAttestation> selectAttestations(final List<LedgerAttestation> all) {
        return List.of();
    }

    @Override
    public Optional<TrustScoreComputer.ActorScore> derive(
            final Map<String, TrustScoreComputer.ActorScore> capabilityScores,
            final List<LedgerAttestation> allAttestations) {

        if (capabilityScores.isEmpty()) {
            return Optional.empty();
        }

        // Count capability-specific attestations per tag (exclude global "*")
        final Map<String, Long> countPerTag = allAttestations.stream()
                .filter(a -> !CapabilityTag.GLOBAL.equals(a.capabilityTag))
                .collect(Collectors.groupingBy(a -> a.capabilityTag, Collectors.counting()));

        final long totalCount = countPerTag.values().stream().mapToLong(Long::longValue).sum();
        if (totalCount == 0) {
            return Optional.empty();
        }

        double weightedScore = 0.0;
        // Alpha and beta are combined proportionally; priors (1.0) are added once, not per-capability
        double combinedAlpha = 1.0;
        double combinedBeta = 1.0;
        int totalDecisions = 0;
        int totalOverturned = 0;
        int totalPositive = 0;
        int totalNegative = 0;

        for (final Map.Entry<String, TrustScoreComputer.ActorScore> entry : capabilityScores.entrySet()) {
            final long count = countPerTag.getOrDefault(entry.getKey(), 0L);
            if (count == 0) {
                continue;
            }
            final double weight = (double) count / totalCount;
            final TrustScoreComputer.ActorScore cs = entry.getValue();
            weightedScore += weight * cs.trustScore();
            combinedAlpha += weight * (cs.alpha() - 1.0);
            combinedBeta += weight * (cs.beta() - 1.0);
            totalDecisions += cs.decisionCount();
            totalOverturned += cs.overturnedCount();
            totalPositive += cs.attestationPositive();
            totalNegative += cs.attestationNegative();
        }

        final double clampedScore = Math.max(0.0, Math.min(1.0, weightedScore));
        return Optional.of(new TrustScoreComputer.ActorScore(
                clampedScore, combinedAlpha, combinedBeta,
                totalDecisions, totalOverturned, totalPositive, totalNegative));
    }
}
```

- [ ] **Step 2: Run all GlobalScoreStrategyTest tests to verify they all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=GlobalScoreStrategyTest -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS, 11 tests PASSED.

- [ ] **Step 3: Commit all three strategy implementations**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/AllAttestationsGlobalStrategy.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/ExplicitGlobalAttestationsStrategy.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/FrequencyWeightedGlobalStrategy.java \
  runtime/src/test/java/io/casehub/ledger/service/GlobalScoreStrategyTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat: add three GlobalScoreStrategy implementations (Options A, B, C)

- AllAttestationsGlobalStrategy (@DefaultBean) — Wang & Vassileva root-node model
- ExplicitGlobalAttestationsStrategy (@Alternative) — only '*' attestations
- FrequencyWeightedGlobalStrategy (@Alternative) — Fan et al. (2015) frequency-weighted

11 unit tests, all passing.

Refs #61 · Part of epic #50"
```

---

### Task 6: Failing integration tests for `TrustScoreJob` capability pass (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityIT.java`

The test profile `trust-score-test` is already configured in `application.properties`:
- `%trust-score-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:trustscoretestdb;...`
- `%trust-score-test.casehub.ledger.trust-score.enabled=true`

- [ ] **Step 1: Write the failing integration test class**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * End-to-end integration tests for capability-scoped trust scoring (issue #61).
 *
 * <p>Covers: happy path, correctness/isolation, robustness, and backward compatibility.
 */
@QuarkusTest
@TestProfile(TrustScoreIT.TrustScoreTestProfile.class)
class TrustScoreCapabilityIT {

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    TrustGateService trustGateService;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    EntityManager em;

    // ── Happy path: capability rows created per distinct tag ──────────────────

    @Test
    @Transactional
    void capabilityTaggedAttestations_createCapabilityRows() {
        final String actorId = "agent-capability-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(3, ChronoUnit.DAYS), AttestationVerdict.FLAGGED, "style-review");

        trustScoreJob.runComputation();

        final List<ActorTrustScore> capScores = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
        assertThat(capScores).hasSize(2);

        final ActorTrustScore secScore = capScores.stream()
                .filter(s -> "security-review".equals(s.scopeKey)).findFirst().orElseThrow();
        assertThat(secScore.trustScore).isGreaterThan(0.6);

        final ActorTrustScore styleScore = capScores.stream()
                .filter(s -> "style-review".equals(s.scopeKey)).findFirst().orElseThrow();
        assertThat(styleScore.trustScore).isLessThan(0.5);
    }

    // ── Happy path: global row uses all attestations (Option B default) ───────

    @Test
    @Transactional
    void globalRow_usesAllAttestations_optionB() {
        final String actorId = "agent-global-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // 2 SOUND security-review + 1 FLAGGED style-review
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(3, ChronoUnit.DAYS), AttestationVerdict.FLAGGED, "style-review");

        trustScoreJob.runComputation();

        final ActorTrustScore global = trustRepo.findByActorId(actorId).orElseThrow();
        final List<ActorTrustScore> caps = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);

        final double secScore = caps.stream().filter(s -> "security-review".equals(s.scopeKey))
                .findFirst().orElseThrow().trustScore;
        final double styleScore = caps.stream().filter(s -> "style-review".equals(s.scopeKey))
                .findFirst().orElseThrow().trustScore;

        // Global must be between style (low) and security (high) scores since it includes all
        assertThat(global.trustScore).isBetween(styleScore, secScore);
        assertThat(global.scoreType).isEqualTo(ScoreType.GLOBAL);
    }

    // ── Correctness: capability Beta is isolated to its own tag ───────────────

    @Test
    @Transactional
    void capabilityScore_isolatedToItsTag() {
        final String actorId = "agent-isolated-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // All security-review: SOUND + SOUND (expect high score ~0.75)
        // All style-review: FLAGGED (expect low score <0.5)
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(3, ChronoUnit.DAYS), AttestationVerdict.FLAGGED, "style-review");

        trustScoreJob.runComputation();

        final var secScore = trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "security-review")
                .orElseThrow();
        final var styleScore = trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "style-review")
                .orElseThrow();

        // Security Beta only from 2 SOUND: α ≈ 3, β ≈ 1, score ≈ 0.75
        assertThat(secScore.trustScore).isCloseTo(0.75, within(0.1));
        // Style Beta only from 1 FLAGGED: α ≈ 1, β > 1, score < 0.5
        assertThat(styleScore.trustScore).isLessThan(0.5);
        // Confirm they differ — isolation is working
        assertThat(secScore.trustScore).isGreaterThan(styleScore.trustScore);
    }

    // ── Correctness: actor with only global ("*") attestations gets no capability rows

    @Test
    @Transactional
    void globalOnlyActor_noCapabilityRows() {
        final String actorId = "agent-star-only-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // All attestations are global "*" (pre-B1 style or explicitly unscoped)
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, CapabilityTag.GLOBAL);
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, CapabilityTag.GLOBAL);

        trustScoreJob.runComputation();

        final List<ActorTrustScore> capScores = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
        assertThat(capScores).isEmpty();

        final ActorTrustScore global = trustRepo.findByActorId(actorId).orElseThrow();
        assertThat(global.trustScore).isGreaterThan(0.5);
    }

    // ── Correctness: TrustGateService uses capability score ───────────────────

    @Test
    @Transactional
    void trustGateService_usesCapabilityScore() {
        final String actorId = "agent-gate-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // 3 SOUND security-review → high capability score
        // 1 FLAGGED style-review → low capability score
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(3, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(4, ChronoUnit.DAYS), AttestationVerdict.FLAGGED, "style-review");

        trustScoreJob.runComputation();

        // High threshold for security-review → passes (capability score ≈ 0.8)
        assertThat(trustGateService.meetsThreshold(actorId, "security-review", 0.7)).isTrue();
        // High threshold for style-review → fails (capability score < 0.5)
        assertThat(trustGateService.meetsThreshold(actorId, "style-review", 0.6)).isFalse();
    }

    // ── Correctness: unknown tag falls back to global ─────────────────────────

    @Test
    @Transactional
    void trustGateService_unknownTag_fallsBackToGlobal() {
        final String actorId = "agent-fallback-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, "security-review");

        trustScoreJob.runComputation();

        final ActorTrustScore global = trustRepo.findByActorId(actorId).orElseThrow();

        // Query for a tag that has no CAPABILITY row → falls back to global
        final boolean result = trustGateService.meetsThreshold(actorId, "nonexistent-capability",
                global.trustScore - 0.1);
        assertThat(result).isTrue();
    }

    // ── Robustness: actor with no attestations → GLOBAL = 0.5; no CAPABILITY rows

    @Test
    @Transactional
    void noAttestations_globalNeutral_noCapabilityRows() {
        final String actorId = "agent-none-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // Decision with no attestation (null verdict)
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), null, CapabilityTag.GLOBAL);

        trustScoreJob.runComputation();

        final ActorTrustScore global = trustRepo.findByActorId(actorId).orElseThrow();
        assertThat(global.trustScore).isCloseTo(0.5, within(0.01));

        final List<ActorTrustScore> caps = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
        assertThat(caps).isEmpty();
    }

    // ── Backward compatibility: pre-B1 actors (all "*") unchanged ────────────

    @Test
    @Transactional
    void preB1Actor_allGlobalAttestations_globalUnchanged() {
        final String actorId = "agent-legacy-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // Simulate pre-B1: all attestations have capabilityTag = "*" (the B1 default)
        seedWithCapability(actorId, now.minus(1, ChronoUnit.DAYS), AttestationVerdict.SOUND, CapabilityTag.GLOBAL);
        seedWithCapability(actorId, now.minus(2, ChronoUnit.DAYS), AttestationVerdict.SOUND, CapabilityTag.GLOBAL);
        seedWithCapability(actorId, now.minus(3, ChronoUnit.DAYS), AttestationVerdict.ENDORSED, CapabilityTag.GLOBAL);

        trustScoreJob.runComputation();

        // Global score should still be high — same as before B2
        final ActorTrustScore global = trustRepo.findByActorId(actorId).orElseThrow();
        assertThat(global.trustScore).isGreaterThan(0.7);

        // No CAPABILITY rows written for pre-B1 actors
        assertThat(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY)).isEmpty();
    }

    // ── Fixtures ─────────────────────────────────────────────────────────────

    private void seedWithCapability(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdictOrNull, final String capabilityTag) {
        LedgerTestFixtures.seedDecision(actorId, decisionTime, verdictOrNull,
                verdictOrNull != null ? decisionTime.plusSeconds(60) : null,
                capabilityTag, repo, em);
    }
}
```

- [ ] **Step 2: Run to verify it fails (TrustScoreJob doesn't compute capability rows yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=TrustScoreCapabilityIT -q 2>&1 | tail -20
```
Expected: FAIL — `capabilityRows` is empty; `findByActorIdAndScoreType` returns nothing.

- [ ] **Step 3: Commit the failing tests**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test: add failing integration tests for capability-scoped trust scoring

TDD — tests fail until TrustScoreJob implements capability pass.
Refs #61 · Part of epic #50"
```

---

### Task 7: `TrustScoreJob` — capability pass + strategy integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Add `GlobalScoreStrategy` injection and capability pass**

The current per-actor loop in `runComputation()` computes only GLOBAL scores. Replace the entire per-actor loop body. Also add the `@Inject` field and required imports.

Add this field after the existing `@Inject` fields:
```java
    @Inject
    GlobalScoreStrategy globalScoreStrategy;
```

Add these imports (after the existing ones):
```java
import java.util.ArrayList;
import java.util.HashSet;
import java.util.LinkedHashMap;

import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.service.GlobalScoreStrategy;
```

Replace the for-loop in `runComputation()` — the entire block from `for (final Map.Entry<String, List<LedgerEntry>> actorEntry : byActor.entrySet()) {` through its closing `}` — with:

```java
        for (final Map.Entry<String, List<LedgerEntry>> actorEntry : byActor.entrySet()) {
            final String actorId = actorEntry.getKey();
            final List<LedgerEntry> decisions = actorEntry.getValue();
            final ActorType actorType = decisions.stream()
                    .map(e -> e.actorType)
                    .filter(t -> t != null)
                    .findFirst()
                    .orElse(ActorType.HUMAN);

            // Collect all attestations for this actor's decisions
            final List<LedgerAttestation> actorAttestations = new ArrayList<>();
            for (final LedgerEntry decision : decisions) {
                actorAttestations.addAll(attestationsByEntry.getOrDefault(decision.id, List.of()));
            }

            // ── Capability pass ────────────────────────────────────────────────────────
            // Group by capabilityTag, excluding the global sentinel "*"
            final Map<String, List<LedgerAttestation>> byCapability = actorAttestations.stream()
                    .filter(a -> !CapabilityTag.GLOBAL.equals(a.capabilityTag))
                    .collect(Collectors.groupingBy(a -> a.capabilityTag));

            final Map<String, TrustScoreComputer.ActorScore> capabilityScores = new LinkedHashMap<>();

            for (final Map.Entry<String, List<LedgerAttestation>> capEntry : byCapability.entrySet()) {
                final String capabilityTag = capEntry.getKey();

                // Filter attestationsByEntry to only this capability tag
                final Map<UUID, List<LedgerAttestation>> capByEntry = attestationsByEntry.entrySet().stream()
                        .collect(Collectors.toMap(
                                Map.Entry::getKey,
                                e -> e.getValue().stream()
                                        .filter(a -> capabilityTag.equals(a.capabilityTag))
                                        .collect(Collectors.toList())));

                final TrustScoreComputer.ActorScore capScore = computer.compute(decisions, capByEntry, now);
                trustRepo.upsert(actorId, ActorTrustScore.ScoreType.CAPABILITY, capabilityTag,
                        actorType, capScore.trustScore(),
                        capScore.decisionCount(), capScore.overturnedCount(),
                        capScore.alpha(), capScore.beta(),
                        capScore.attestationPositive(), capScore.attestationNegative(), now);
                capabilityScores.put(capabilityTag, capScore);
            }

            // ── Global pass ────────────────────────────────────────────────────────────
            final List<LedgerAttestation> selectedAttestations =
                    globalScoreStrategy.selectAttestations(actorAttestations);
            final Set<LedgerAttestation> selectedSet = new HashSet<>(selectedAttestations);

            final Map<UUID, List<LedgerAttestation>> selectedByEntry = attestationsByEntry.entrySet().stream()
                    .collect(Collectors.toMap(
                            Map.Entry::getKey,
                            e -> e.getValue().stream()
                                    .filter(selectedSet::contains)
                                    .collect(Collectors.toList())));

            final TrustScoreComputer.ActorScore globalScore = computer.compute(decisions, selectedByEntry, now);
            final TrustScoreComputer.ActorScore finalScore =
                    globalScoreStrategy.derive(capabilityScores, actorAttestations)
                            .orElse(globalScore);

            trustRepo.upsert(actorId, ActorTrustScore.ScoreType.GLOBAL, null,
                    actorType, finalScore.trustScore(),
                    finalScore.decisionCount(), finalScore.overturnedCount(),
                    finalScore.alpha(), finalScore.beta(),
                    finalScore.attestationPositive(), finalScore.attestationNegative(), now);
        }
```

- [ ] **Step 2: Check IntelliJ for errors on the modified file**

Use `mcp__intellij__get_file_problems` with `filePath=runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`.
Expected: no errors.

- [ ] **Step 3: Run TrustScoreCapabilityIT to verify the new tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=TrustScoreCapabilityIT -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS, 8 tests PASSED.

- [ ] **Step 4: Run the existing TrustScoreIT to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=TrustScoreIT -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS, all existing tests PASSED.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat: extend TrustScoreJob to compute per-capability Beta trust scores

- Capability pass before global (required for FrequencyWeightedGlobalStrategy.derive())
- GlobalScoreStrategy injected; default AllAttestationsGlobalStrategy used
- Backward-compatible: actors with only '*' attestations get no CAPABILITY rows

Refs #61 · Part of epic #50"
```

---

### Task 8: `TrustGateService` Phase 2 — unit tests + implementation

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java`

- [ ] **Step 1: Add Phase 2 unit tests to `TrustGateServiceTest`**

The existing `StubRepository` has `findByActorIdAndTypeAndKey` returning `Optional.empty()`. Extend the stub to support capability scores. Read the current file first via IntelliJ (`mcp__intellij__get_file_text_by_path`).

Add a factory method for creating a stub with both global AND capability scores:

```java
    private static ActorTrustScoreRepository repoWith(
            final String actorId, final double globalScore,
            final String capabilityTag, final double capabilityScore) {
        final ActorTrustScore global = new ActorTrustScore();
        global.id = UUID.randomUUID();
        global.actorId = actorId;
        global.scoreType = ScoreType.GLOBAL;
        global.actorType = ActorType.AGENT;
        global.trustScore = globalScore;
        global.lastComputedAt = Instant.now();

        final ActorTrustScore capability = new ActorTrustScore();
        capability.id = UUID.randomUUID();
        capability.actorId = actorId;
        capability.scoreType = ScoreType.CAPABILITY;
        capability.scopeKey = capabilityTag;
        capability.actorType = ActorType.AGENT;
        capability.trustScore = capabilityScore;
        capability.lastComputedAt = Instant.now();

        return new StubRepository(global) {
            @Override
            public Optional<ActorTrustScore> findByActorIdAndTypeAndKey(
                    final String id, final ScoreType type, final String scopeKey) {
                if (actorId.equals(id) && ScoreType.CAPABILITY.equals(type)
                        && capabilityTag.equals(scopeKey)) {
                    return Optional.of(capability);
                }
                return Optional.empty();
            }
        };
    }
```

Add these test methods to the class (in the `meetsThreshold (capability overload)` section, replacing the existing Phase 1 test):

```java
    // ── meetsThreshold (capability overload — Phase 2) ────────────────────────

    @Test
    void meetsThreshold_withCapability_usesCapabilityScore_whenAvailable() {
        // global = 0.4 (would fail), capability "security-review" = 0.9 (should pass)
        final TrustGateService gate = new TrustGateService(
                repoWith("actor-x", 0.4, "security-review", 0.9));

        assertThat(gate.meetsThreshold("actor-x", "security-review", 0.8)).isTrue();
    }

    @Test
    void meetsThreshold_withCapability_capabilityScoreBelowThreshold() {
        // global = 0.9 (would pass), capability "style-review" = 0.3 (should fail)
        final TrustGateService gate = new TrustGateService(
                repoWith("actor-y", 0.9, "style-review", 0.3));

        assertThat(gate.meetsThreshold("actor-y", "style-review", 0.8)).isFalse();
    }

    @Test
    void meetsThreshold_withCapability_fallsBackToGlobal_whenNoCapabilityScore() {
        // No capability row for "unknown-tag" → falls back to global = 0.85
        final TrustGateService gate = new TrustGateService(repoWith("actor-z", 0.85));

        assertThat(gate.meetsThreshold("actor-z", "unknown-tag", 0.8)).isTrue();
        assertThat(gate.meetsThreshold("actor-z", "unknown-tag", 0.9)).isFalse();
    }

    @Test
    void meetsThreshold_withCapability_falseWhenNoScoreAtAll() {
        final TrustGateService gate = new TrustGateService(emptyRepo());

        assertThat(gate.meetsThreshold("ghost", "security-review", 0.0)).isFalse();
    }

    // ── currentScore (capability overload) ───────────────────────────────────

    @Test
    void currentScore_withCapability_returnsCapabilityScore() {
        final TrustGateService gate = new TrustGateService(
                repoWith("actor-q", 0.5, "security-review", 0.9));

        assertThat(gate.currentScore("actor-q", "security-review")).isPresent();
        assertThat(gate.currentScore("actor-q", "security-review").get()).isEqualTo(0.9);
    }

    @Test
    void currentScore_withCapability_emptyWhenNoCapabilityScore() {
        final TrustGateService gate = new TrustGateService(repoWith("actor-r", 0.7));

        assertThat(gate.currentScore("actor-r", "unknown-tag")).isEmpty();
    }
```

- [ ] **Step 2: Run to verify the new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=TrustGateServiceTest -q 2>&1 | tail -10
```
Expected: FAIL — `meetsThreshold(actorId, capabilityTag, minTrust)` still falls through to global; `currentScore(actorId, capabilityTag)` method doesn't exist.

- [ ] **Step 3: Implement Phase 2 in `TrustGateService`**

Replace the `meetsThreshold(actorId, capabilityTag, minTrust)` method and add `currentScore(actorId, capabilityTag)`. Read the current file first via IntelliJ.

Replace the stub method:
```java
    public boolean meetsThreshold(final String actorId, final String capabilityTag,
            final double minTrust) {
        return repository
                .findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, capabilityTag)
                .map(s -> s.trustScore >= minTrust)
                .orElseGet(() -> meetsThreshold(actorId, minTrust));
    }
```

Add after `currentScore(actorId)`:
```java
    /**
     * Returns the actor's trust score for the given capability, or {@link Optional#empty()} if
     * no capability-specific score has been computed yet.
     *
     * @param actorId the actor identity string
     * @param capabilityTag the capability tag (e.g. {@code "security-review"})
     * @return the capability trust score in [0.0, 1.0], or empty
     */
    public Optional<Double> currentScore(final String actorId, final String capabilityTag) {
        return repository
                .findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, capabilityTag)
                .map(s -> s.trustScore);
    }
```

Also add the import at the top:
```java
import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
```

- [ ] **Step 4: Check IntelliJ for errors**

`mcp__intellij__get_file_problems` on `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java`.

- [ ] **Step 5: Run TrustGateServiceTest to verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=TrustGateServiceTest -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS, all tests PASSED (original tests + 6 new Phase 2 tests).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java \
  runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat: TrustGateService Phase 2 — capability score lookup with global fallback

meetsThreshold(actorId, capabilityTag, minTrust) now queries CAPABILITY score
first, falls back to GLOBAL when none exists. currentScore(actorId, capabilityTag)
added. 6 new unit tests.

Refs #61 · Part of epic #50"
```

---

### Task 9: Full build verification

- [ ] **Step 1: Build and test all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean install -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 2: Check test count increased**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -3
```
Report the new total (was 247; expect ~270+ with new unit + IT tests).

---

### Task 10: ADR + Documentation update

**Files:**
- Create: `adr/0008-global-score-strategy-spi.md`
- Modify: `docs/DESIGN.md`
- Modify: `docs/DESIGN-capabilities.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Create ADR 0008**

```markdown
# ADR 0008 — GlobalScoreStrategy SPI

**Date:** 2026-05-02
**Status:** Accepted
**Refs:** #61

## Context

`TrustScoreJob` needed to compute per-capability Beta scores using `capabilityTag` (added in #60).
A key design question was what the GLOBAL `ActorTrustScore` should aggregate:

- **Option A:** Only `capabilityTag = "*"` attestations
- **Option B:** All attestations regardless of tag (holistic view)
- **Option C:** Frequency-weighted combination of capability scores

Research across three papers found genuine disagreement:
- Wang & Vassileva (2003) — global is the root node computed from all interactions (Option B)
- Fan et al. (2015) — global = weighted combination of dimension-specific scores (Option C)
- Semantic separation argument — global = explicit cross-capability statements (Option A)

## Decision

Introduce `GlobalScoreStrategy` as a CDI SPI with two methods:
1. `selectAttestations(all)` — filter attestations before the global Beta model
2. `derive(capabilityScores, all)` — optionally override the Beta result after capability scores are computed

Three built-in implementations:
- `AllAttestationsGlobalStrategy` — `@DefaultBean`; Option B (Wang & Vassileva)
- `ExplicitGlobalAttestationsStrategy` — `@Alternative`; Option A
- `FrequencyWeightedGlobalStrategy` — `@Alternative`; Option C (Fan et al.)

Default = Option B. Rationale: simplest, no dependency on capability scores being computed
first, consistent with the root-node model in Wang & Vassileva.

## Consequences

- `TrustScoreJob` injects `GlobalScoreStrategy`; capability scores are computed before global
  (required for Option C's `derive()` call)
- Consumers switch strategies via `quarkus.arc.selected-alternatives` — no code change
- Option C (`FrequencyWeightedGlobalStrategy`) is available as an `@Alternative` but not the
  default; production deployments with real data can evaluate whether frequency weighting
  better reflects holistic trust in their context
- Follow-up: revisit default after real deployment data is available

## References

- Wang & Vassileva (2003): [Semantic Scholar](https://www.semanticscholar.org/paper/Bayesian-Network-Based-Trust-Model-in-Peer-to-Peer-Wang-Vassileva/bed571c548e1d858f4acd7e351289b28dd56de7e)
- Fan et al. (2015): DOI [10.1007/s11633-014-0840-3](https://doi.org/10.1007/s11633-014-0840-3)
- Jøsang & Ismail (2002): [BLED 2002](https://aisel.aisnet.org/bled2002/41/)
```

Use `mcp__intellij__create_new_file` with `pathInProject=adr/0008-global-score-strategy-spi.md` and the content above.

- [ ] **Step 2: Update `adr/INDEX.md`**

Add row:
```markdown
| [0008](0008-global-score-strategy-spi.md) | GlobalScoreStrategy SPI — pluggable global trust score aggregation | Accepted | 2026-05-02 |
```

- [ ] **Step 3: Update `docs/DESIGN.md` — Implementation Tracker**

Add a new row after the `capabilityTag on LedgerAttestation` row:
```markdown
| **Capability-scoped trust scores** | ✅ Done | `GlobalScoreStrategy` SPI (3 implementations: all-attestations default, explicit-global, frequency-weighted); `TrustScoreJob` capability pass; `TrustGateService` Phase 2 (capability-then-global fallback). ADR 0008. Closes #61. |
```

Also update the Roadmap near-term trust scoring paragraph — find the text `TrustScoreJob writes GLOBAL rows; capability and dimension computation is added by #61 and #62 respectively` and update:
```
TrustScoreJob writes GLOBAL rows (✅ #60) and CAPABILITY rows (✅ #61); dimension computation is added by #62.
```

- [ ] **Step 4: Update `docs/DESIGN-capabilities.md` — Trust Scoring section**

Find the `## Trust Scoring — Capability Tags` section added in #60. After the table of SPI query methods, add:

```markdown
## Trust Scoring — Capability Beta Scores

`TrustScoreJob` computes per-capability Beta models alongside the global score (✅ #61).
For each distinct `capabilityTag ≠ "*"` in the actor's attestations, a separate `CAPABILITY`
row is written to `actor_trust_score` with `scope_key = capabilityTag`.

The global score aggregation strategy is pluggable via `GlobalScoreStrategy` (see ADR 0008):

| Implementation | CDI | Behaviour | Backed by |
|---|---|---|---|
| `AllAttestationsGlobalStrategy` | `@DefaultBean` | All attestations → global Beta | Wang & Vassileva (2003) |
| `ExplicitGlobalAttestationsStrategy` | `@Alternative` | Only `"*"` attestations → global Beta | Semantic separation |
| `FrequencyWeightedGlobalStrategy` | `@Alternative` | Global = Σ(count_i/total × capScore_i) | Fan et al. (2015) |

`TrustGateService.meetsThreshold(actorId, capabilityTag, minTrust)` queries the CAPABILITY
score first and falls back to GLOBAL when none exists.

---
```

- [ ] **Step 5: Update `CLAUDE.md` — project structure**

In the `service/` section, add the four new files after `TrustGateService.java`:
```
│           ├── GlobalScoreStrategy.java          — SPI: select attestations / derive global trust score
│           ├── AllAttestationsGlobalStrategy.java — @DefaultBean: all attestations → global Beta (Option B)
│           ├── ExplicitGlobalAttestationsStrategy.java — @Alternative: only "*" attestations (Option A)
│           ├── FrequencyWeightedGlobalStrategy.java — @Alternative: frequency-weighted from capability scores (Option C)
```

- [ ] **Step 6: Search for stale cross-references using IntelliJ**

Use `mcp__intellij-index__ide_search_text` with query `#61` to find any remaining TODO/pending references to #61 in docs or code. Fix any found.

Also search for `TrustGateService` to check any Javadoc or doc references still say "Phase 1" — update to remove the phase language.

- [ ] **Step 7: Commit documentation**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  adr/0008-global-score-strategy-spi.md \
  adr/INDEX.md \
  docs/DESIGN.md \
  docs/DESIGN-capabilities.md \
  CLAUDE.md
git -C /Users/mdproctor/claude/casehub/ledger commit -m "docs: document capability-scoped trust scoring across ADR, DESIGN, CLAUDE.md

- ADR 0008: GlobalScoreStrategy SPI with literature references
- DESIGN.md tracker: #61 row added
- DESIGN-capabilities.md: capability Beta scores + strategy table
- CLAUDE.md: four new strategy classes in project structure

Closes #61 · Part of epic #50"
```

---

### Task 11: Close issue

- [ ] **Step 1: Push to remote**

```bash
git -C /Users/mdproctor/claude/casehub/ledger push origin main 2>&1 | tail -3
```

- [ ] **Step 2: Close the issue with a summary**

```bash
gh issue close 61 --repo casehubio/ledger --comment "Implemented across commits on main.

**What landed:**
- \`GlobalScoreStrategy\` SPI with 3 implementations: \`AllAttestationsGlobalStrategy\` (@DefaultBean, Option B), \`ExplicitGlobalAttestationsStrategy\` (@Alternative, Option A), \`FrequencyWeightedGlobalStrategy\` (@Alternative, Option C)
- \`TrustScoreJob\` capability pass: per-capability Beta models computed before global; strategy injected
- \`TrustGateService\` Phase 2: capability-then-global fallback; \`currentScore(actorId, capabilityTag)\`
- ADR 0008: GlobalScoreStrategy SPI with full literature references
- 11 unit tests + 8 IT tests

Closes #61 · Refs epic #50"
```

- [ ] **Step 3: Verify epic #50 reflects #61 closed**

```bash
gh issue view 50 --repo casehubio/ledger | head -20
```
