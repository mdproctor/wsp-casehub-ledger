# Trust Federation and Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Also required:** Invoke `superpowers:test-driven-development` before implementing each task. Invoke `java-dev` for all Java work (loads testing-principles + ide-tooling). Invoke `superpowers:requesting-code-review` before committing. Invoke `implementation-doc-sync` after all tasks complete.

**Goal:** Add `TrustExportService` (structured trust read-model), `TrustImportService` SPI (pluggable import strategy), and `TrustBootstrapService` (first-registration seeding hook) to casehub-ledger.

**Architecture:** All new classes live under `runtime/service/federation/`. `TrustExportService` reads `ActorTrustScore` rows and projects them into a structured payload. `TrustImportService` is a SPI with a no-op default and a JPA seed-if-absent alternative. `TrustBootstrapService` is called from a batch pre-pass at the start of `TrustScoreJob.runComputation()` for actors with no existing trust score.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JPA/Hibernate, AssertJ, JUnit 5, H2 (tests).

**Issues:** #63 (C1 TrustExportService), #64 (C2 TrustImportService), #65 (D1 bootstrap). Epics #51, #52.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
**Single-test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ClassName"`

---

## File Map

**New — `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/`**
- `TrustExportPayload.java` — record: exportedAt, exportingDeployment, actors
- `ActorExport.java` — record: actorId, actorType, globalScore, capabilityScores, dimensionScores
- `GlobalScoreExport.java` — record: alpha, beta, trustScore, decisionCount, attestationPositive, attestationNegative, lastComputedAt
- `CapabilityScoreExport.java` — record: capabilityTag + same fields as GlobalScoreExport
- `DimensionScoreExport.java` — record: dimension, score, sampleCount, lastComputedAt
- `TrustExportService.java` — CDI bean: exportAll, exportActor, exportDelta
- `TrustImportService.java` — SPI interface: importTrust(TrustExportPayload)
- `NoOpTrustImportService.java` — @DefaultBean, empty body
- `JpaTrustImportService.java` — @Alternative, seed-if-absent
- `TrustBootstrapSource.java` — SPI interface: fetchPriorTrust(actorId)
- `NoOpTrustBootstrapSource.java` — @DefaultBean, returns Optional.empty()
- `TrustBootstrapService.java` — CDI bean: bootstrapIfNew(Set<String> newActorIds)

**Modified**
- `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java` — add @NamedQuery for findAllByLastComputedAtAfter
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java` — add findAllByLastComputedAtAfter(Instant)
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java` — implement findAllByLastComputedAtAfter
- `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java` — add ExportConfig and BootstrapConfig nested interfaces
- `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` — add bootstrap pre-pass before actor loop
- `runtime/src/test/resources/application.properties` — add federation-export-test, federation-import-test, federation-bootstrap-test profiles

**New tests — `runtime/src/test/java/io/casehub/ledger/service/federation/`**
- `TrustExportServiceIT.java` — @QuarkusTest, profile federation-export-test
- `TrustImportServiceIT.java` — @QuarkusTest, profile federation-import-test
- `TrustBootstrapServiceIT.java` — @QuarkusTest, profile federation-bootstrap-test
- `TrustBootstrapSourceTest.java` — plain JUnit 5

---

## Task 1: Config additions

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Step 1: Add ExportConfig and BootstrapConfig interfaces to TrustScoreConfig**

  Inside `LedgerConfig.TrustScoreConfig`, add after the existing `EigenTrustConfig` interface:

  ```java
  /**
   * Trust export settings — deployment identity for exported payloads.
   *
   * @return the export sub-configuration
   */
  ExportConfig export();

  /**
   * Trust bootstrap settings — seed Beta(α,β) from an external source on first registration.
   *
   * @return the bootstrap sub-configuration
   */
  BootstrapConfig bootstrap();

  /** Trust export settings. */
  interface ExportConfig {

      /**
       * Opaque identifier for this deployment included in exported trust payloads.
       * Informational only — consumers may use it to filter out their own exports when
       * importing from an aggregator. Empty string by default.
       *
       * @return deployment identifier, or empty string if not configured
       */
      @WithDefault("")
      String deploymentId();
  }

  /** Trust bootstrap settings. */
  interface BootstrapConfig {

      /**
       * When {@code true}, a batch pre-pass at the start of each {@link TrustScoreJob}
       * run calls {@link io.casehub.ledger.runtime.service.federation.TrustBootstrapService}
       * for actors with no existing trust score. Off by default — zero overhead when disabled.
       *
       * @return {@code true} if trust bootstrapping is enabled; {@code false} by default
       */
      @WithDefault("false")
      boolean enabled();
  }
  ```

- [ ] **Step 2: Verify it compiles**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ActorTrustScoreRepositoryIT"
  ```

  Expected: BUILD SUCCESS (existing test passes, config additions are inert).

- [ ] **Step 3: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
  git commit -m "feat(#63): add ExportConfig and BootstrapConfig to LedgerConfig"
  ```

---

## Task 2: Export payload value types + repository delta query

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportPayload.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/ActorExport.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/GlobalScoreExport.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/CapabilityScoreExport.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/DimensionScoreExport.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`

- [ ] **Step 1: Create the five record types**

  **`TrustExportPayload.java`:**
  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;
  import java.util.List;

  /** Snapshot of trust scores for one or more actors, shaped for dashboard and cross-deployment use. */
  public record TrustExportPayload(
          Instant exportedAt,
          String exportingDeployment,
          List<ActorExport> actors) {
  }
  ```

  **`ActorExport.java`:**
  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.util.List;

  import io.casehub.ledger.api.model.ActorType;

  /** All trust scores for a single actor, structured by score type. */
  public record ActorExport(
          String actorId,
          ActorType actorType,
          GlobalScoreExport globalScore,
          List<CapabilityScoreExport> capabilityScores,
          List<DimensionScoreExport> dimensionScores) {
  }
  ```

  **`GlobalScoreExport.java`:**
  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;

  /** Bayesian Beta global trust score for an actor. */
  public record GlobalScoreExport(
          double alpha,
          double beta,
          double trustScore,
          int decisionCount,
          int attestationPositive,
          int attestationNegative,
          Instant lastComputedAt) {
  }
  ```

  **`CapabilityScoreExport.java`:**
  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;

  /** Bayesian Beta trust score for an actor scoped to a single capability tag. */
  public record CapabilityScoreExport(
          String capabilityTag,
          double alpha,
          double beta,
          double trustScore,
          int decisionCount,
          int attestationPositive,
          int attestationNegative,
          Instant lastComputedAt) {
  }
  ```

  **`DimensionScoreExport.java`:**
  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;

  /**
   * Continuous quality dimension score for an actor.
   *
   * <p>
   * {@code sampleCount} is {@code attestationPositive + attestationNegative} from the source row.
   * The positive/negative split is not preserved — DIMENSION scores are approximated as priors
   * and recomputed from raw attestations on the next {@link io.casehub.ledger.runtime.service.TrustScoreJob} run.
   */
  public record DimensionScoreExport(
          String dimension,
          double score,
          int sampleCount,
          Instant lastComputedAt) {
  }
  ```

- [ ] **Step 2: Add NamedQuery to ActorTrustScore**

  In `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`, add after the last existing `@NamedQuery`:

  ```java
  @NamedQuery(name = "ActorTrustScore.findAllByLastComputedAtAfter",
          query = "SELECT s FROM ActorTrustScore s WHERE s.lastComputedAt > :since")
  ```

- [ ] **Step 3: Add method to ActorTrustScoreRepository SPI**

  In `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java`, add after `findAll()`:

  ```java
  /**
   * Return all trust scores (across all actors and score types) whose
   * {@code lastComputedAt} timestamp is strictly after {@code since}.
   * Used by {@link io.casehub.ledger.runtime.service.federation.TrustExportService#exportDelta}.
   */
  List<ActorTrustScore> findAllByLastComputedAtAfter(Instant since);
  ```

- [ ] **Step 4: Implement in JpaActorTrustScoreRepository**

  In `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`, add after `findAll()`:

  ```java
  @Override
  public List<ActorTrustScore> findAllByLastComputedAtAfter(final Instant since) {
      return em.createNamedQuery("ActorTrustScore.findAllByLastComputedAtAfter", ActorTrustScore.class)
              .setParameter("since", since)
              .getResultList();
  }
  ```

- [ ] **Step 5: Verify all compiles and existing tests pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ActorTrustScoreRepositoryIT"
  ```

  Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/ \
          runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java \
          runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java \
          runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java
  git commit -m "feat(#63): add TrustExportPayload types and findAllByLastComputedAtAfter repository method"
  ```

---

## Task 3: TrustExportService (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustExportServiceIT.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Add federation-export-test profile to application.properties**

  Append to `runtime/src/test/resources/application.properties`:

  ```properties
  # Federation export test profile (used by TrustExportServiceIT)
  # Isolated DB — prevents actor_id collisions with other test classes
  # trust-score enabled so upsert paths work (no scheduler needed — direct repo writes)
  %federation-export-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:federationexporttestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
  %federation-export-test.casehub.ledger.trust-score.enabled=true
  %federation-export-test.quarkus.scheduler.enabled=false
  ```

- [ ] **Step 2: Write the failing test class**

  Create `runtime/src/test/java/io/casehub/ledger/service/federation/TrustExportServiceIT.java`:

  ```java
  package io.casehub.ledger.service.federation;

  import static org.assertj.core.api.Assertions.assertThat;

  import java.time.Instant;

  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;

  import org.junit.jupiter.api.Test;

  import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
  import io.casehub.ledger.api.model.ActorType;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import io.casehub.ledger.runtime.service.federation.TrustExportService;
  import io.quarkus.test.junit.QuarkusTest;
  import io.quarkus.test.junit.QuarkusTestProfile;
  import io.quarkus.test.junit.TestProfile;

  @QuarkusTest
  @TestProfile(TrustExportServiceIT.ExportTestProfile.class)
  class TrustExportServiceIT {

      public static class ExportTestProfile implements QuarkusTestProfile {
          @Override
          public String getConfigProfile() {
              return "federation-export-test";
          }
      }

      @Inject TrustExportService exportService;
      @Inject ActorTrustScoreRepository trustRepo;

      // ── exportAll ─────────────────────────────────────────────────────────

      @Test
      @Transactional
      void exportAll_returnsActorsAtOrAboveThreshold() {
          final String high = "export-high-" + System.nanoTime();
          final String low  = "export-low-"  + System.nanoTime();
          final Instant now = Instant.now();

          trustRepo.upsert(high, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.85, 10, 1, 8.0, 2.0, 9, 1, now);
          trustRepo.upsert(low, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.30, 5, 3, 2.0, 3.0, 2, 3, now);

          final var payload = exportService.exportAll(0.5);

          assertThat(payload.actors()).extracting(a -> a.actorId()).contains(high);
          assertThat(payload.actors()).extracting(a -> a.actorId()).doesNotContain(low);
      }

      @Test
      @Transactional
      void exportAll_zeroThreshold_returnsAllActorsWithGlobalScore() {
          final String a = "export-zero-a-" + System.nanoTime();
          final String b = "export-zero-b-" + System.nanoTime();
          final Instant now = Instant.now();

          trustRepo.upsert(a, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.1, 2, 1, 1.0, 2.0, 1, 1, now);
          trustRepo.upsert(b, ScoreType.GLOBAL, null, ActorType.HUMAN,
                  0.9, 8, 0, 7.0, 1.0, 8, 0, now);

          final var payload = exportService.exportAll(0.0);

          assertThat(payload.actors()).extracting(a2 -> a2.actorId()).contains(a, b);
      }

      @Test
      @Transactional
      void exportAll_excludesActorsWithNoGlobalRow() {
          final String capOnly = "export-caponly-" + System.nanoTime();
          final Instant now = Instant.now();

          trustRepo.upsert(capOnly, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                  0.9, 5, 0, 4.0, 1.0, 5, 0, now);

          final var payload = exportService.exportAll(0.0);

          assertThat(payload.actors()).extracting(a -> a.actorId()).doesNotContain(capOnly);
      }

      // ── exportActor ───────────────────────────────────────────────────────

      @Test
      @Transactional
      void exportActor_returnsStructuredExportWithAllScoreTypes() {
          final String actorId = "export-actor-" + System.nanoTime();
          final Instant now = Instant.now();

          trustRepo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.80, 10, 1, 8.0, 2.0, 9, 1, now);
          trustRepo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                  0.90, 6, 0, 5.0, 1.0, 6, 0, now);
          trustRepo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                  0.75, 6, 0, 0.0, 0.0, 5, 1, now);

          final var result = exportService.exportActor(actorId);

          assertThat(result).isPresent();
          final var export = result.get();
          assertThat(export.actors()).hasSize(1);

          final var actor = export.actors().get(0);
          assertThat(actor.actorId()).isEqualTo(actorId);
          assertThat(actor.actorType()).isEqualTo(ActorType.AGENT);

          assertThat(actor.globalScore()).isNotNull();
          assertThat(actor.globalScore().trustScore()).isEqualTo(0.80);
          assertThat(actor.globalScore().alpha()).isEqualTo(8.0);

          assertThat(actor.capabilityScores()).hasSize(1);
          assertThat(actor.capabilityScores().get(0).capabilityTag()).isEqualTo("security-review");
          assertThat(actor.capabilityScores().get(0).trustScore()).isEqualTo(0.90);

          assertThat(actor.dimensionScores()).hasSize(1);
          assertThat(actor.dimensionScores().get(0).dimension()).isEqualTo("thoroughness");
          assertThat(actor.dimensionScores().get(0).score()).isEqualTo(0.75);
          assertThat(actor.dimensionScores().get(0).sampleCount()).isEqualTo(6);
      }

      @Test
      @Transactional
      void exportActor_returnsEmpty_forUnknownActor() {
          assertThat(exportService.exportActor("no-such-actor-" + System.nanoTime())).isEmpty();
      }

      // ── exportDelta ───────────────────────────────────────────────────────

      @Test
      @Transactional
      void exportDelta_returnsActorsWithScoresChangedAfterSince() {
          final Instant since = Instant.now().minusSeconds(60);
          final String changed = "export-delta-new-"  + System.nanoTime();
          final String stable  = "export-delta-old-"  + System.nanoTime();

          trustRepo.upsert(stable, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.7, 5, 0, 4.0, 1.0, 5, 0, since.minusSeconds(10));
          trustRepo.upsert(changed, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.8, 8, 1, 6.0, 2.0, 7, 1, since.plusSeconds(10));

          final var payload = exportService.exportDelta(since);

          assertThat(payload.actors()).extracting(a -> a.actorId()).contains(changed);
          assertThat(payload.actors()).extracting(a -> a.actorId()).doesNotContain(stable);
      }

      @Test
      @Transactional
      void exportDelta_returnsEmptyActors_whenNoScoresChangedSince() {
          final Instant since = Instant.now().plusSeconds(60);
          final String actorId = "export-delta-none-" + System.nanoTime();
          final Instant now = Instant.now();

          trustRepo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.7, 5, 0, 4.0, 1.0, 5, 0, now);

          final var payload = exportService.exportDelta(since);

          assertThat(payload.actors()).isEmpty();
      }

      // ── payload metadata ──────────────────────────────────────────────────

      @Test
      @Transactional
      void exportAll_payloadContainsExportedAtTimestamp() {
          final Instant before = Instant.now().minusSeconds(1);
          final var payload = exportService.exportAll(0.0);
          assertThat(payload.exportedAt()).isAfter(before);
      }
  }
  ```

- [ ] **Step 3: Run test — verify it fails with class not found**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustExportServiceIT"
  ```

  Expected: COMPILATION FAILURE — `TrustExportService` does not exist yet.

- [ ] **Step 4: Implement TrustExportService**

  Create `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java`:

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;
  import java.util.ArrayList;
  import java.util.List;
  import java.util.Map;
  import java.util.Optional;
  import java.util.Set;
  import java.util.stream.Collectors;

  import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
  import io.casehub.ledger.api.model.ActorType;
  import io.casehub.ledger.runtime.config.LedgerConfig;
  import io.casehub.ledger.runtime.model.ActorTrustScore;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;

  /**
   * Structured read-model over {@link ActorTrustScore}.
   *
   * <p>
   * Consumed by upper layers (dashboard, compliance reports) and future cross-deployment
   * trust federation. See design spec 2026-05-12-trust-federation-bootstrap-design.md.
   */
  @ApplicationScoped
  public class TrustExportService {

      @Inject
      ActorTrustScoreRepository trustRepo;

      @Inject
      LedgerConfig config;

      /**
       * Export all actors whose GLOBAL trust score meets or exceeds {@code minTrustScore}.
       * Actors with no GLOBAL row are excluded regardless of threshold.
       */
      public TrustExportPayload exportAll(final double minTrustScore) {
          final List<ActorTrustScore> all = trustRepo.findAll();
          final Set<String> qualifying = all.stream()
                  .filter(s -> s.scoreType == ScoreType.GLOBAL && s.trustScore >= minTrustScore)
                  .map(s -> s.actorId)
                  .collect(Collectors.toSet());
          final List<ActorTrustScore> scores = all.stream()
                  .filter(s -> qualifying.contains(s.actorId))
                  .collect(Collectors.toList());
          return buildPayload(scores);
      }

      /**
       * Export a single actor's complete trust profile.
       *
       * @return empty if the actor has no computed trust scores
       */
      public Optional<TrustExportPayload> exportActor(final String actorId) {
          final List<ActorTrustScore> scores = new ArrayList<>();
          scores.addAll(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.GLOBAL));
          scores.addAll(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY));
          scores.addAll(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION));
          if (scores.isEmpty()) {
              return Optional.empty();
          }
          return Optional.of(buildPayload(scores));
      }

      /**
       * Export complete profiles for all actors with any score change after {@code since}.
       * Returns an empty actors list if no scores have changed.
       */
      public TrustExportPayload exportDelta(final Instant since) {
          final List<ActorTrustScore> all = trustRepo.findAll();
          final Set<String> changedActors = all.stream()
                  .filter(s -> s.lastComputedAt != null && s.lastComputedAt.isAfter(since))
                  .map(s -> s.actorId)
                  .collect(Collectors.toSet());
          final List<ActorTrustScore> scores = all.stream()
                  .filter(s -> changedActors.contains(s.actorId))
                  .collect(Collectors.toList());
          return buildPayload(scores);
      }

      private TrustExportPayload buildPayload(final List<ActorTrustScore> scores) {
          final Map<String, List<ActorTrustScore>> byActor = scores.stream()
                  .collect(Collectors.groupingBy(s -> s.actorId));
          final List<ActorExport> actors = byActor.values().stream()
                  .map(this::toActorExport)
                  .collect(Collectors.toList());
          return new TrustExportPayload(
                  Instant.now(),
                  config.trustScore().export().deploymentId(),
                  actors);
      }

      private ActorExport toActorExport(final List<ActorTrustScore> scores) {
          final String actorId = scores.get(0).actorId;
          final ActorType actorType = scores.stream()
                  .map(s -> s.actorType)
                  .filter(t -> t != null)
                  .findFirst()
                  .orElse(ActorType.HUMAN);

          final GlobalScoreExport global = scores.stream()
                  .filter(s -> s.scoreType == ScoreType.GLOBAL)
                  .findFirst()
                  .map(s -> new GlobalScoreExport(s.alpha, s.beta, s.trustScore,
                          s.decisionCount, s.attestationPositive, s.attestationNegative,
                          s.lastComputedAt))
                  .orElse(null);

          final List<CapabilityScoreExport> capabilities = scores.stream()
                  .filter(s -> s.scoreType == ScoreType.CAPABILITY)
                  .map(s -> new CapabilityScoreExport(s.scopeKey, s.alpha, s.beta, s.trustScore,
                          s.decisionCount, s.attestationPositive, s.attestationNegative,
                          s.lastComputedAt))
                  .collect(Collectors.toList());

          final List<DimensionScoreExport> dimensions = scores.stream()
                  .filter(s -> s.scoreType == ScoreType.DIMENSION)
                  .map(s -> new DimensionScoreExport(s.scopeKey, s.trustScore,
                          s.attestationPositive + s.attestationNegative, s.lastComputedAt))
                  .collect(Collectors.toList());

          return new ActorExport(actorId, actorType, global, capabilities, dimensions);
      }
  }
  ```

- [ ] **Step 5: Run tests — verify they pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustExportServiceIT"
  ```

  Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 6: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java \
          runtime/src/test/java/io/casehub/ledger/service/federation/TrustExportServiceIT.java \
          runtime/src/test/resources/application.properties
  git commit -m "feat(#63): TrustExportService — structured trust read-model (exportAll/exportActor/exportDelta)"
  ```

---

## Task 4: TrustImportService SPI and NoOpTrustImportService

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustImportService.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/NoOpTrustImportService.java`

No test for the interface itself. The no-op is tested implicitly via `TrustImportServiceIT` in Task 5.

- [ ] **Step 1: Create TrustImportService interface**

  ```java
  package io.casehub.ledger.runtime.service.federation;

  /**
   * SPI for importing trust scores from an external {@link TrustExportPayload}.
   *
   * <p>
   * The implementation is the strategy — different implementations embody different merge
   * behaviours (seed-if-absent, weighted average, replace). Consumers provide a custom
   * implementation as a CDI {@code @Alternative} when the built-in
   * {@link JpaTrustImportService} does not suit their merge policy.
   *
   * <p>
   * Default: {@link NoOpTrustImportService} — no scores are written.
   * Built-in alternative: {@link JpaTrustImportService} — seed-if-absent for all score types.
   */
  public interface TrustImportService {

      /**
       * Import trust scores from the given payload into this deployment.
       * Implementations decide how to handle conflicts with existing scores.
       *
       * @param payload the trust scores to import
       */
      void importTrust(TrustExportPayload payload);
  }
  ```

- [ ] **Step 2: Create NoOpTrustImportService**

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import io.quarkus.arc.DefaultBean;
  import jakarta.enterprise.context.ApplicationScoped;

  /** Default no-op — trust import is opt-in. Activate {@link JpaTrustImportService} or provide a custom implementation. */
  @DefaultBean
  @ApplicationScoped
  public class NoOpTrustImportService implements TrustImportService {

      @Override
      public void importTrust(final TrustExportPayload payload) {
          // no-op
      }
  }
  ```

- [ ] **Step 3: Verify compilation**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustExportServiceIT"
  ```

  Expected: BUILD SUCCESS (existing tests still pass).

- [ ] **Step 4: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustImportService.java \
          runtime/src/main/java/io/casehub/ledger/runtime/service/federation/NoOpTrustImportService.java
  git commit -m "feat(#64): TrustImportService SPI + NoOpTrustImportService @DefaultBean"
  ```

---

## Task 5: JpaTrustImportService (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustImportServiceIT.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Add federation-import-test profile to application.properties**

  Append to `runtime/src/test/resources/application.properties`:

  ```properties
  # Federation import test profile (used by TrustImportServiceIT)
  # Isolated DB + JpaTrustImportService activated as selected alternative
  %federation-import-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:federationimporttestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
  %federation-import-test.quarkus.arc.selected-alternatives=\
    io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
    io.casehub.ledger.runtime.service.federation.JpaTrustImportService
  %federation-import-test.quarkus.scheduler.enabled=false
  ```

- [ ] **Step 2: Write the failing test**

  Create `runtime/src/test/java/io/casehub/ledger/service/federation/TrustImportServiceIT.java`:

  ```java
  package io.casehub.ledger.service.federation;

  import static org.assertj.core.api.Assertions.assertThat;

  import java.time.Instant;
  import java.util.List;

  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;

  import org.junit.jupiter.api.Test;

  import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
  import io.casehub.ledger.api.model.ActorType;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import io.casehub.ledger.runtime.service.federation.ActorExport;
  import io.casehub.ledger.runtime.service.federation.CapabilityScoreExport;
  import io.casehub.ledger.runtime.service.federation.DimensionScoreExport;
  import io.casehub.ledger.runtime.service.federation.GlobalScoreExport;
  import io.casehub.ledger.runtime.service.federation.TrustExportPayload;
  import io.casehub.ledger.runtime.service.federation.TrustImportService;
  import io.quarkus.test.junit.QuarkusTest;
  import io.quarkus.test.junit.QuarkusTestProfile;
  import io.quarkus.test.junit.TestProfile;

  @QuarkusTest
  @TestProfile(TrustImportServiceIT.ImportTestProfile.class)
  class TrustImportServiceIT {

      public static class ImportTestProfile implements QuarkusTestProfile {
          @Override
          public String getConfigProfile() {
              return "federation-import-test";
          }
      }

      @Inject TrustImportService importService;
      @Inject ActorTrustScoreRepository trustRepo;

      // ── seed-if-absent: new actor gets all rows ───────────────────────────

      @Test
      @Transactional
      void importTrust_seedsNewActor_allScoreTypes() {
          final String actorId = "import-new-" + System.nanoTime();
          final Instant ts = Instant.now();

          final var payload = payloadFor(actorId,
                  new GlobalScoreExport(8.0, 2.0, 0.80, 10, 9, 1, ts),
                  List.of(new CapabilityScoreExport("security-review", 5.0, 1.0, 0.83, 6, 6, 0, ts)),
                  List.of(new DimensionScoreExport("thoroughness", 0.75, 5, ts)));

          importService.importTrust(payload);

          assertThat(trustRepo.findByActorId(actorId)).isPresent();
          assertThat(trustRepo.findByActorId(actorId).get().trustScore).isEqualTo(0.80);

          final var caps = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
          assertThat(caps).hasSize(1);
          assertThat(caps.get(0).scopeKey).isEqualTo("security-review");

          final var dims = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION);
          assertThat(dims).hasSize(1);
          assertThat(dims.get(0).scopeKey).isEqualTo("thoroughness");
          assertThat(dims.get(0).trustScore).isEqualTo(0.75);
      }

      // ── seed-if-absent: existing actor is untouched ───────────────────────

      @Test
      @Transactional
      void importTrust_skipsExistingActor_whenGlobalRowPresent() {
          final String actorId = "import-existing-" + System.nanoTime();
          final Instant ts = Instant.now();

          trustRepo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.50, 3, 1, 2.0, 1.0, 2, 1, ts);

          final var payload = payloadFor(actorId,
                  new GlobalScoreExport(8.0, 2.0, 0.80, 10, 9, 1, ts),
                  List.of(), List.of());

          importService.importTrust(payload);

          assertThat(trustRepo.findByActorId(actorId).get().trustScore).isEqualTo(0.50);
      }

      // ── mixed payload: seeds new, skips existing ──────────────────────────

      @Test
      @Transactional
      void importTrust_mixedPayload_onlyNewActorSeeded() {
          final String existing = "import-mixed-old-" + System.nanoTime();
          final String fresh    = "import-mixed-new-" + System.nanoTime();
          final Instant ts = Instant.now();

          trustRepo.upsert(existing, ScoreType.GLOBAL, null, ActorType.AGENT,
                  0.60, 5, 0, 3.0, 2.0, 5, 0, ts);

          final var existingExport = actorExport(existing,
                  new GlobalScoreExport(9.0, 1.0, 0.90, 10, 10, 0, ts), List.of(), List.of());
          final var freshExport = actorExport(fresh,
                  new GlobalScoreExport(7.0, 3.0, 0.70, 10, 7, 3, ts), List.of(), List.of());

          importService.importTrust(new TrustExportPayload(Instant.now(), "", List.of(existingExport, freshExport)));

          assertThat(trustRepo.findByActorId(existing).get().trustScore).isEqualTo(0.60);
          assertThat(trustRepo.findByActorId(fresh).get().trustScore).isEqualTo(0.70);
      }

      // ── no-op when payload is empty ───────────────────────────────────────

      @Test
      @Transactional
      void importTrust_emptyPayload_writesNothing() {
          final var payload = new TrustExportPayload(Instant.now(), "", List.of());
          importService.importTrust(payload);
          // no assertion needed beyond "no exception"
      }

      // ── helpers ───────────────────────────────────────────────────────────

      private TrustExportPayload payloadFor(final String actorId,
              final GlobalScoreExport global,
              final List<CapabilityScoreExport> caps,
              final List<DimensionScoreExport> dims) {
          return new TrustExportPayload(Instant.now(), "",
                  List.of(actorExport(actorId, global, caps, dims)));
      }

      private ActorExport actorExport(final String actorId, final GlobalScoreExport global,
              final List<CapabilityScoreExport> caps, final List<DimensionScoreExport> dims) {
          return new ActorExport(actorId, ActorType.AGENT, global, caps, dims);
      }
  }
  ```

- [ ] **Step 3: Run — verify compilation failure**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustImportServiceIT"
  ```

  Expected: COMPILATION FAILURE — `JpaTrustImportService` does not exist.

- [ ] **Step 4: Implement JpaTrustImportService**

  Create `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java`:

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.time.Instant;

  import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.enterprise.inject.Alternative;
  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;

  /**
   * Seed-if-absent import strategy.
   *
   * <p>
   * For each actor in the payload: if a GLOBAL row already exists for that actor, skip
   * the entire actor. Otherwise, write GLOBAL, CAPABILITY, and DIMENSION rows from the export.
   * This ensures bootstrap seeds are written once and never overwrite locally-computed scores.
   *
   * <p>
   * Activate via:
   * {@code quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.service.federation.JpaTrustImportService}
   * (alongside any other selected alternatives already configured).
   *
   * <p>
   * For custom merge behaviour, implement {@link TrustImportService} directly.
   */
  @Alternative
  @ApplicationScoped
  public class JpaTrustImportService implements TrustImportService {

      @Inject
      ActorTrustScoreRepository trustRepo;

      @Override
      @Transactional
      public void importTrust(final TrustExportPayload payload) {
          final Instant now = Instant.now();
          for (final ActorExport actor : payload.actors()) {
              if (trustRepo.findByActorId(actor.actorId()).isPresent()) {
                  continue;
              }
              seedActor(actor, now);
          }
      }

      private void seedActor(final ActorExport actor, final Instant now) {
          if (actor.globalScore() != null) {
              final GlobalScoreExport g = actor.globalScore();
              trustRepo.upsert(actor.actorId(), ScoreType.GLOBAL, null,
                      actor.actorType(), g.trustScore(),
                      g.decisionCount(), 0, g.alpha(), g.beta(),
                      g.attestationPositive(), g.attestationNegative(), now);
          }
          for (final CapabilityScoreExport c : actor.capabilityScores()) {
              trustRepo.upsert(actor.actorId(), ScoreType.CAPABILITY, c.capabilityTag(),
                      actor.actorType(), c.trustScore(),
                      c.decisionCount(), 0, c.alpha(), c.beta(),
                      c.attestationPositive(), c.attestationNegative(), now);
          }
          for (final DimensionScoreExport d : actor.dimensionScores()) {
              trustRepo.upsert(actor.actorId(), ScoreType.DIMENSION, d.dimension(),
                      actor.actorType(), d.score(),
                      d.sampleCount(), 0, 0.0, 0.0,
                      d.sampleCount(), 0, now);
          }
      }
  }
  ```

- [ ] **Step 5: Run tests — verify they pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustImportServiceIT"
  ```

  Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 6: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java \
          runtime/src/test/java/io/casehub/ledger/service/federation/TrustImportServiceIT.java \
          runtime/src/test/resources/application.properties
  git commit -m "feat(#64): JpaTrustImportService — seed-if-absent @Alternative for TrustImportService"
  ```

---

## Task 6: TrustBootstrapSource SPI, NoOp, and plain unit test

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustBootstrapSource.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/NoOpTrustBootstrapSource.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapSourceTest.java`

- [ ] **Step 1: Create TrustBootstrapSource interface**

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.util.Optional;

  /**
   * SPI for fetching prior trust data for an actor from an external source.
   *
   * <p>
   * Called by {@link TrustBootstrapService} the first time an actor appears in this deployment.
   * Implementations may call a remote deployment, query a shared registry, or any other source.
   *
   * <p>
   * Default: {@link NoOpTrustBootstrapSource} — returns empty, preserving Beta(1,1) uninformative prior.
   */
  public interface TrustBootstrapSource {

      /**
       * Fetch prior trust data for the given actor, if available.
       *
       * @param actorId the actor identifier to look up
       * @return a payload containing the actor's prior scores, or empty if none available
       */
      Optional<TrustExportPayload> fetchPriorTrust(String actorId);
  }
  ```

- [ ] **Step 2: Create NoOpTrustBootstrapSource**

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.util.Optional;

  import io.quarkus.arc.DefaultBean;
  import jakarta.enterprise.context.ApplicationScoped;

  /** Default no-op — trust bootstrapping is opt-in. Provide a custom {@link TrustBootstrapSource} to activate. */
  @DefaultBean
  @ApplicationScoped
  public class NoOpTrustBootstrapSource implements TrustBootstrapSource {

      @Override
      public Optional<TrustExportPayload> fetchPriorTrust(final String actorId) {
          return Optional.empty();
      }
  }
  ```

- [ ] **Step 3: Write and run plain unit test**

  Create `runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapSourceTest.java`:

  ```java
  package io.casehub.ledger.service.federation;

  import static org.assertj.core.api.Assertions.assertThat;

  import org.junit.jupiter.api.Test;

  import io.casehub.ledger.runtime.service.federation.NoOpTrustBootstrapSource;

  class TrustBootstrapSourceTest {

      @Test
      void noOp_alwaysReturnsEmpty() {
          final var source = new NoOpTrustBootstrapSource();
          assertThat(source.fetchPriorTrust("claude:analyst@v1")).isEmpty();
          assertThat(source.fetchPriorTrust("user-123")).isEmpty();
          assertThat(source.fetchPriorTrust("")).isEmpty();
      }
  }
  ```

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustBootstrapSourceTest"
  ```

  Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustBootstrapSource.java \
          runtime/src/main/java/io/casehub/ledger/runtime/service/federation/NoOpTrustBootstrapSource.java \
          runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapSourceTest.java
  git commit -m "feat(#65): TrustBootstrapSource SPI + NoOpTrustBootstrapSource @DefaultBean"
  ```

---

## Task 7: TrustBootstrapService (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapServiceIT.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustBootstrapService.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Add federation-bootstrap-test profile**

  Append to `runtime/src/test/resources/application.properties`:

  ```properties
  # Federation bootstrap test profile (used by TrustBootstrapServiceIT)
  # Isolated DB + JpaTrustImportService active so seeded rows are written
  %federation-bootstrap-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:federationbootstraptestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
  %federation-bootstrap-test.quarkus.arc.selected-alternatives=\
    io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
    io.casehub.ledger.runtime.service.federation.JpaTrustImportService
  %federation-bootstrap-test.quarkus.scheduler.enabled=false
  ```

- [ ] **Step 2: Write the failing test**

  Create `runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapServiceIT.java`:

  ```java
  package io.casehub.ledger.service.federation;

  import static org.assertj.core.api.Assertions.assertThat;

  import java.time.Instant;
  import java.util.ArrayList;
  import java.util.List;
  import java.util.Optional;
  import java.util.Set;

  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.enterprise.inject.Alternative;
  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;

  import org.junit.jupiter.api.BeforeEach;
  import org.junit.jupiter.api.Test;

  import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
  import io.casehub.ledger.api.model.ActorType;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import io.casehub.ledger.runtime.service.federation.ActorExport;
  import io.casehub.ledger.runtime.service.federation.GlobalScoreExport;
  import io.casehub.ledger.runtime.service.federation.TrustBootstrapService;
  import io.casehub.ledger.runtime.service.federation.TrustBootstrapSource;
  import io.casehub.ledger.runtime.service.federation.TrustExportPayload;
  import io.quarkus.test.junit.QuarkusTest;
  import io.quarkus.test.junit.QuarkusTestProfile;
  import io.quarkus.test.junit.TestProfile;

  @QuarkusTest
  @TestProfile(TrustBootstrapServiceIT.BootstrapTestProfile.class)
  class TrustBootstrapServiceIT {

      public static class BootstrapTestProfile implements QuarkusTestProfile {
          @Override
          public String getConfigProfile() {
              return "federation-bootstrap-test";
          }
      }

      /** Inner @Alternative replaces NoOpTrustBootstrapSource in this test context. */
      @Alternative
      @ApplicationScoped
      static class CapturingBootstrapSource implements TrustBootstrapSource {
          static final List<String> QUERIED = new ArrayList<>();
          static TrustExportPayload RESPONSE = null;

          @Override
          public Optional<TrustExportPayload> fetchPriorTrust(final String actorId) {
              QUERIED.add(actorId);
              return Optional.ofNullable(RESPONSE);
          }
      }

      @BeforeEach
      void reset() {
          CapturingBootstrapSource.QUERIED.clear();
          CapturingBootstrapSource.RESPONSE = null;
      }

      @Inject TrustBootstrapService bootstrapService;
      @Inject ActorTrustScoreRepository trustRepo;

      // ── new actor + source returns payload → rows seeded ─────────────────

      @Test
      @Transactional
      void bootstrapIfNew_seedsNewActor_whenSourceReturnsPayload() {
          final String actorId = "bootstrap-new-" + System.nanoTime();
          final Instant ts = Instant.now();

          final var global = new GlobalScoreExport(7.0, 3.0, 0.70, 10, 7, 3, ts);
          final var actor = new ActorExport(actorId, ActorType.AGENT, global, List.of(), List.of());
          CapturingBootstrapSource.RESPONSE = new TrustExportPayload(ts, "", List.of(actor));

          bootstrapService.bootstrapIfNew(Set.of(actorId));

          assertThat(CapturingBootstrapSource.QUERIED).contains(actorId);
          assertThat(trustRepo.findByActorId(actorId)).isPresent();
          assertThat(trustRepo.findByActorId(actorId).get().trustScore).isEqualTo(0.70);
      }

      // ── new actor + source returns empty → no rows written ────────────────

      @Test
      @Transactional
      void bootstrapIfNew_writesNothing_whenSourceReturnsEmpty() {
          final String actorId = "bootstrap-empty-" + System.nanoTime();
          CapturingBootstrapSource.RESPONSE = null;

          bootstrapService.bootstrapIfNew(Set.of(actorId));

          assertThat(CapturingBootstrapSource.QUERIED).contains(actorId);
          assertThat(trustRepo.findByActorId(actorId)).isEmpty();
      }

      // ── empty set → source never called ──────────────────────────────────

      @Test
      void bootstrapIfNew_emptySet_doesNothing() {
          bootstrapService.bootstrapIfNew(Set.of());
          assertThat(CapturingBootstrapSource.QUERIED).isEmpty();
      }
  }
  ```

- [ ] **Step 3: Run — verify compilation failure**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustBootstrapServiceIT"
  ```

  Expected: COMPILATION FAILURE — `TrustBootstrapService` does not exist.

- [ ] **Step 4: Implement TrustBootstrapService**

  Create `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustBootstrapService.java`:

  ```java
  package io.casehub.ledger.runtime.service.federation;

  import java.util.Set;

  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;

  /**
   * Seeds trust scores for actors appearing in this deployment for the first time.
   *
   * <p>
   * Called from {@link io.casehub.ledger.runtime.service.TrustScoreJob} as a batch pre-pass
   * before the per-actor computation loop. Only fires when
   * {@code casehub.ledger.trust-score.bootstrap.enabled=true}.
   */
  @ApplicationScoped
  public class TrustBootstrapService {

      @Inject
      TrustBootstrapSource bootstrapSource;

      @Inject
      TrustImportService importService;

      /**
       * For each actor ID in {@code newActorIds}, fetch prior trust from the configured source
       * and import it via {@link TrustImportService}. Actors for which the source returns empty
       * are silently skipped — they start from Beta(1,1).
       *
       * @param newActorIds actor IDs with no existing {@link io.casehub.ledger.runtime.model.ActorTrustScore} row
       */
      public void bootstrapIfNew(final Set<String> newActorIds) {
          for (final String actorId : newActorIds) {
              bootstrapSource.fetchPriorTrust(actorId)
                      .ifPresent(importService::importTrust);
          }
      }
  }
  ```

- [ ] **Step 5: Activate CapturingBootstrapSource in test beans.xml**

  The `@Alternative` inner class must be listed in `META-INF/beans.xml`. Open `runtime/src/test/resources/META-INF/beans.xml` and verify it contains an `<alternatives>` section or the mode is `annotated`. In Quarkus, `@Alternative` inner classes in `@QuarkusTest` classes are auto-discovered if CDI scanning includes the test class. Verify by running the tests — if CDI rejects the alternative, add:

  ```xml
  <alternatives>
    <class>io.casehub.ledger.service.federation.TrustBootstrapServiceIT$CapturingBootstrapSource</class>
  </alternatives>
  ```

- [ ] **Step 6: Run tests — verify they pass**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustBootstrapServiceIT"
  ```

  Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 7: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustBootstrapService.java \
          runtime/src/test/java/io/casehub/ledger/service/federation/TrustBootstrapServiceIT.java \
          runtime/src/test/resources/application.properties
  git commit -m "feat(#65): TrustBootstrapService — fetches prior trust and seeds via TrustImportService"
  ```

---

## Task 8: Bootstrap pre-pass in TrustScoreJob

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreIT.java` (or equivalent existing job test)

- [ ] **Step 1: Familiarise yourself with the existing test helper**

  The existing job tests use `LedgerTestFixtures.seedDecision()` to persist `TestEntry` records with optional attestations. Find it at:
  `runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java`

  Signature used in this task:
  ```java
  LedgerTestFixtures.seedDecision(actorId, Instant.now(), null, repo, em);
  // actorId: String, decisionTime: Instant, verdict: null (unattested), repo, em
  ```

  The relevant existing test class is:
  `runtime/src/test/java/io/casehub/ledger/service/TrustScoreIT.java`

- [ ] **Step 2: Add bootstrap pre-pass to TrustScoreJob**

  In `TrustScoreJob`, add to imports:

  ```java
  import io.casehub.ledger.runtime.service.federation.TrustBootstrapService;
  ```

  Add injection after existing `@Inject` fields:

  ```java
  @Inject
  TrustBootstrapService bootstrapService;
  ```

  Add to the import block at the top of the file (alongside existing imports):

  ```java
  import java.util.LinkedHashSet;
  import java.util.Set;
  import io.casehub.ledger.runtime.service.federation.TrustBootstrapService;
  ```

  Add the pre-pass at the start of `runComputation()`, immediately after the `previousSnapshot` block and before the `TrustScoreComputer computer = ...` line:

  ```java
  if (config.trustScore().bootstrap().enabled()) {
      final Set<String> existingActors = trustRepo.findAll().stream()
              .map(s -> s.actorId)
              .collect(Collectors.toSet());
      final Set<String> newActors = new LinkedHashSet<>(byActor.keySet());
      newActors.removeAll(existingActors);
      if (!newActors.isEmpty()) {
          bootstrapService.bootstrapIfNew(newActors);
      }
  }
  ```

- [ ] **Step 3: Add a test that verifies bootstrap disabled by default**

  In the existing `TrustScoreIT.java` (trust-score-test profile), add:

  ```java
  @Test
  @Transactional
  void runComputation_bootstrapDisabledByDefault_doesNotCallBootstrapSource() {
      // The default config has bootstrap.enabled=false.
      // Run the job — if bootstrap fired, it would call NoOpTrustBootstrapSource which is safe.
      // Verify the job completes without error when there are no ledger entries.
      job.runComputation();
      // No entries → no actors → no trust scores written → success
      assertThat(trustRepo.findAll()).isEmpty();
  }
  ```

- [ ] **Step 4: Add bootstrap-enabled test to a new profile**

  Add to `runtime/src/test/resources/application.properties`:

  ```properties
  # Trust score bootstrap integration test profile
  %trust-score-bootstrap-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:trustscorebootstraptestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
  %trust-score-bootstrap-test.casehub.ledger.trust-score.enabled=true
  %trust-score-bootstrap-test.casehub.ledger.trust-score.bootstrap.enabled=true
  %trust-score-bootstrap-test.quarkus.arc.selected-alternatives=\
    io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
    io.casehub.ledger.runtime.service.federation.JpaTrustImportService
  %trust-score-bootstrap-test.quarkus.scheduler.enabled=false
  ```

  Create `runtime/src/test/java/io/casehub/ledger/service/TrustScoreBootstrapIT.java`:

  ```java
  package io.casehub.ledger.service;

  import static org.assertj.core.api.Assertions.assertThat;

  import java.time.Instant;
  import java.util.ArrayList;
  import java.util.List;
  import java.util.Optional;
  import java.util.UUID;

  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.enterprise.inject.Alternative;
  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;

  import org.junit.jupiter.api.BeforeEach;
  import org.junit.jupiter.api.Test;

  import jakarta.persistence.EntityManager;

  import io.casehub.ledger.api.model.ActorType;
  import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
  import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
  import io.casehub.ledger.runtime.service.TrustScoreJob;
  import io.casehub.ledger.runtime.service.federation.ActorExport;
  import io.casehub.ledger.runtime.service.federation.GlobalScoreExport;
  import io.casehub.ledger.runtime.service.federation.TrustBootstrapSource;
  import io.casehub.ledger.runtime.service.federation.TrustExportPayload;
  import io.casehub.ledger.service.LedgerTestFixtures;
  import io.quarkus.test.junit.QuarkusTest;
  import io.quarkus.test.junit.QuarkusTestProfile;
  import io.quarkus.test.junit.TestProfile;

  @QuarkusTest
  @TestProfile(TrustScoreBootstrapIT.BootstrapJobProfile.class)
  class TrustScoreBootstrapIT {

      public static class BootstrapJobProfile implements QuarkusTestProfile {
          @Override
          public String getConfigProfile() {
              return "trust-score-bootstrap-test";
          }
      }

      @Alternative
      @ApplicationScoped
      static class SeedingBootstrapSource implements TrustBootstrapSource {
          static final List<String> QUERIED = new ArrayList<>();
          static double SEED_SCORE = 0.75;

          @Override
          public Optional<TrustExportPayload> fetchPriorTrust(final String actorId) {
              QUERIED.add(actorId);
              final Instant ts = Instant.now();
              final var global = new GlobalScoreExport(6.0, 2.0, SEED_SCORE, 8, 6, 2, ts);
              final var actor = new ActorExport(actorId, ActorType.AGENT, global, List.of(), List.of());
              return Optional.of(new TrustExportPayload(ts, "test-source", List.of(actor)));
          }
      }

      @BeforeEach
      void reset() {
          SeedingBootstrapSource.QUERIED.clear();
          SeedingBootstrapSource.SEED_SCORE = 0.75;
      }

      @Inject TrustScoreJob job;
      @Inject ActorTrustScoreRepository trustRepo;
      @Inject LedgerEntryRepository repo;
      @Inject EntityManager em;

      @Test
      @Transactional
      void runComputation_bootstrapEnabled_seedsNewActorBeforeComputation() {
          final String actorId = "bootstrap-job-" + UUID.randomUUID();

          // Write a ledger entry so the actor appears in byActor map
          LedgerTestFixtures.seedDecision(actorId, Instant.now(), null, repo, em);

          // Actor has no existing trust score — bootstrap should fire
          assertThat(trustRepo.findByActorId(actorId)).isEmpty();

          job.runComputation();

          // Bootstrap was queried
          assertThat(SeedingBootstrapSource.QUERIED).contains(actorId);

          // Score was written (computation ran after seeded row was written)
          assertThat(trustRepo.findByActorId(actorId)).isPresent();
      }

      @Test
      @Transactional
      void runComputation_bootstrapEnabled_existingActorNotQueried() {
          final String actorId = "bootstrap-existing-" + UUID.randomUUID();

          // Pre-seed a trust score so actor is "existing"
          trustRepo.upsert(actorId, io.casehub.ledger.api.model.ActorTrustScore.ScoreType.GLOBAL,
                  null, ActorType.AGENT, 0.60, 5, 0, 3.0, 2.0, 5, 0, Instant.now());

          // Write a ledger entry so actor appears in byActor map
          LedgerTestFixtures.seedDecision(actorId, Instant.now(), null, repo, em);

          job.runComputation();

          // Existing actor was NOT queried for bootstrap
          assertThat(SeedingBootstrapSource.QUERIED).doesNotContain(actorId);
      }
  }
  ```

- [ ] **Step 5: Run all new tests**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustScoreBootstrapIT,TrustScoreIT"
  ```

  Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 6: Run full runtime test suite**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
  ```

  Expected: BUILD SUCCESS. All existing tests still pass.

- [ ] **Step 7: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
          runtime/src/test/java/io/casehub/ledger/service/TrustScoreBootstrapIT.java \
          runtime/src/test/resources/application.properties
  git commit -m "feat(#65): TrustScoreJob — bootstrap pre-pass seeds new actors before computation Closes #65 Refs #63 Refs #64"
  ```

---

## Final: Full build verification

- [ ] **Run complete build**

  ```bash
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
  ```

  Expected: BUILD SUCCESS across all modules (api, runtime, deployment).

- [ ] **Invoke superpowers:requesting-code-review** before the final commit/PR.

- [ ] **Invoke implementation-doc-sync** to check docs drift after all tasks complete.
