# Prerequisite Refactors Implementation Plan (#67 + #68)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement two prerequisite refactors — a pluggable `LedgerEntryEnricher` pipeline (#67) and a discriminator-based `ActorTrustScore` model (#68) — that unblock the full Group A–D trust capability roadmap.

**Architecture:** #67 extracts the existing `LedgerTraceListener` `@PrePersist` into a CDI-injectable `Instance<LedgerEntryEnricher>` pipeline; `LedgerTraceListener` becomes a non-fatal pipeline runner. #68 replaces the single-actor-row model with a `(actor_id, score_type, scope_key)` discriminator keyed by a surrogate UUID, using `UNIQUE NULLS NOT DISTINCT` to handle the nullable `scope_key` for `GLOBAL` rows; all existing GLOBAL behaviour is preserved unchanged.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (JPA named queries), Flyway migrations, H2 (tests), PostgreSQL (prod), JUnit 5, AssertJ, `@QuarkusTest`

**Issue linkage:** All commits must include `Refs #67` (Part 1) or `Refs #68` (Part 2).

---

## Part 1: #67 — LedgerEntry Enrichment Pipeline

### File Map

| Action | File | Purpose |
|---|---|---|
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java` | New SPI interface |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/TraceIdEnricher.java` | Extracts trace-ID logic from LedgerTraceListener |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java` | Becomes non-fatal pipeline runner |
| Modify | `runtime/src/test/java/io/casehub/ledger/service/LedgerTraceListenerIT.java` | Rename comment block; tests must still pass unchanged |
| Create | `runtime/src/test/java/io/casehub/ledger/service/TraceIdEnricherTest.java` | Unit tests for TraceIdEnricher in isolation |
| Create | `runtime/src/test/java/io/casehub/ledger/service/LedgerEnricherPipelineIT.java` | Pipeline non-fatal + multi-enricher integration tests |
| Create | `adr/0005-ledger-entry-enricher-spi.md` | ADR documenting the enricher SPI decision |
| Modify | `docs/DESIGN.md` | Update enricher pipeline description (lines 433, 462) |
| Modify | `CLAUDE.md` | Update project structure table entry for LedgerTraceListener |

---

### Task 1: Create the LedgerEntryEnricher SPI

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java`

- [ ] **Step 1: Create the interface**

```java
package io.casehub.ledger.runtime.service;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * SPI for auto-populating fields on {@link LedgerEntry} at persist time.
 *
 * <p>Implementations are CDI beans discovered via {@code @Inject @Any Instance<LedgerEntryEnricher>}
 * and invoked in the {@code @PrePersist} pipeline. Implementations must be idempotent and
 * non-fatal — a thrown exception is logged and swallowed; the persist is never blocked.
 *
 * <p>Register an implementation by creating an {@code @ApplicationScoped} CDI bean that
 * implements this interface. No registration step is required.
 */
public interface LedgerEntryEnricher {

    /**
     * Enrich the given entry before it is persisted.
     * Called once per {@code @PrePersist} event. Must not throw.
     */
    void enrich(LedgerEntry entry);
}
```

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java
git commit -m "feat(enricher): add LedgerEntryEnricher SPI — pluggable @PrePersist pipeline

Refs #67"
```

---

### Task 2: Write failing unit tests for TraceIdEnricher

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TraceIdEnricherTest.java`

`TraceIdEnricher` will receive an injected `LedgerTraceIdProvider`. Tests run without a Quarkus container using a hand-constructed stub.

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.LedgerTraceIdProvider;
import io.casehub.ledger.runtime.service.TraceIdEnricher;
import io.casehub.ledger.service.supplement.TestEntry;

class TraceIdEnricherTest {

    private static final String TRACE_ID = "abc123";

    private static TestEntry entry() {
        final TestEntry e = new TestEntry();
        e.id = UUID.randomUUID();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "actor";
        e.actorType = ActorType.AGENT;
        e.occurredAt = Instant.now();
        return e;
    }

    private static LedgerTraceIdProvider providing(final String traceId) {
        return () -> Optional.ofNullable(traceId);
    }

    @Test
    void populatesTraceId_whenNullAndProviderHasTrace() {
        final TraceIdEnricher enricher = new TraceIdEnricher(providing(TRACE_ID));
        final TestEntry entry = entry();

        enricher.enrich(entry);

        assertThat(entry.traceId).isEqualTo(TRACE_ID);
    }

    @Test
    void doesNotOverwrite_whenCallerAlreadySetTraceId() {
        final TraceIdEnricher enricher = new TraceIdEnricher(providing(TRACE_ID));
        final TestEntry entry = entry();
        entry.traceId = "caller-supplied";

        enricher.enrich(entry);

        assertThat(entry.traceId).isEqualTo("caller-supplied");
    }

    @Test
    void leavesTraceIdNull_whenProviderReturnsEmpty() {
        final TraceIdEnricher enricher = new TraceIdEnricher(providing(null));
        final TestEntry entry = entry();

        enricher.enrich(entry);

        assertThat(entry.traceId).isNull();
    }
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TraceIdEnricherTest -pl runtime
```

Expected: `COMPILATION ERROR — TraceIdEnricher does not exist`

---

### Task 3: Implement TraceIdEnricher

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/TraceIdEnricher.java`

The constructor injection form (used in unit tests) must coexist with CDI field injection (used at runtime).

- [ ] **Step 1: Create the enricher**

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Enricher that auto-populates {@link LedgerEntry#traceId} from the active OTel span.
 * Extracted from {@code LedgerTraceListener} — same behaviour, now as a pipeline participant.
 */
@ApplicationScoped
public class TraceIdEnricher implements LedgerEntryEnricher {

    private final LedgerTraceIdProvider traceIdProvider;

    @Inject
    public TraceIdEnricher(final LedgerTraceIdProvider traceIdProvider) {
        this.traceIdProvider = traceIdProvider;
    }

    @Override
    public void enrich(final LedgerEntry entry) {
        if (entry.traceId != null) {
            return;
        }
        traceIdProvider.currentTraceId().ifPresent(id -> entry.traceId = id);
    }
}
```

- [ ] **Step 2: Run unit tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=TraceIdEnricherTest -pl runtime
```

Expected: `Tests run: 3, Failures: 0`

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TraceIdEnricher.java \
        runtime/src/test/java/io/casehub/ledger/service/TraceIdEnricherTest.java
git commit -m "feat(enricher): TraceIdEnricher — trace-ID logic extracted from LedgerTraceListener

Refs #67"
```

---

### Task 4: Refactor LedgerTraceListener to pipeline runner

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java`

The class name and `@EntityListeners` registration on `LedgerEntry` stay unchanged. The `@PrePersist` becomes a non-fatal loop over `Instance<LedgerEntryEnricher>`.

- [ ] **Step 1: Rewrite LedgerTraceListener**

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.persistence.PrePersist;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * JPA entity listener that runs the {@link LedgerEntryEnricher} pipeline on every
 * {@link LedgerEntry} before it is persisted.
 *
 * <p>Registered via {@code @EntityListeners} on {@link LedgerEntry}. Enrichers are
 * CDI beans discovered via {@code Instance<LedgerEntryEnricher>}. Each enricher runs
 * in an unspecified order; failure is logged and swallowed — the persist is never blocked.
 */
@ApplicationScoped
public class LedgerTraceListener {

    private static final Logger log = Logger.getLogger(LedgerTraceListener.class);

    @Inject
    @Any
    Instance<LedgerEntryEnricher> enrichers;

    @PrePersist
    public void prePersist(final Object entity) {
        if (!(entity instanceof LedgerEntry entry)) {
            return;
        }
        for (final LedgerEntryEnricher enricher : enrichers) {
            try {
                enricher.enrich(entry);
            } catch (final Exception ex) {
                log.warnf("LedgerEntryEnricher %s failed — entry will still be saved: %s",
                        enricher.getClass().getSimpleName(), ex.getMessage());
            }
        }
    }
}
```

- [ ] **Step 2: Run the full test suite — all existing tests must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: `Tests run: 212, Failures: 0` (same count as before — pure behaviour-preserving refactor)

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java
git commit -m "refactor(enricher): LedgerTraceListener becomes non-fatal enricher pipeline runner

Behaviour-preserving — TraceIdEnricher implements the same logic as before.
Instance<LedgerEntryEnricher> discovers all registered enrichers via CDI.

Refs #67"
```

---

### Task 5: Write and run pipeline integration tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerEnricherPipelineIT.java`

Two behaviours need integration-level verification that unit tests cannot provide:
1. A throwing enricher must not prevent the entry from being saved
2. Multiple enrichers all run (pipeline doesn't short-circuit)

The `ThrowingEnricher` and `CountingEnricher` are declared as package-private static inner classes of the IT — they are CDI-visible only within this test class's Quarkus context by using a dedicated `@TestProfile`. The profile activates a separate H2 database to keep test state isolated.

- [ ] **Step 1: Write the failing integration tests**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicInteger;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerEntryEnricher;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Verifies the LedgerEntryEnricher pipeline properties: non-fatal failure and full execution.
 */
@QuarkusTest
@TestProfile(LedgerEnricherPipelineIT.PipelineTestProfile.class)
class LedgerEnricherPipelineIT {

    public static class PipelineTestProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "enricher-pipeline-test";
        }
    }

    /** Always throws — used to verify the pipeline is non-fatal. */
    @ApplicationScoped
    static class ThrowingEnricher implements LedgerEntryEnricher {
        @Override
        public void enrich(final LedgerEntry entry) {
            throw new RuntimeException("simulated enricher failure");
        }
    }

    /** Counts invocations — used to verify all enrichers run. */
    @ApplicationScoped
    static class CountingEnricher implements LedgerEntryEnricher {
        static final AtomicInteger count = new AtomicInteger(0);

        @Override
        public void enrich(final LedgerEntry entry) {
            count.incrementAndGet();
        }
    }

    @Inject
    LedgerEntryRepository repo;

    @BeforeEach
    void reset() {
        CountingEnricher.count.set(0);
    }

    @Test
    @Transactional
    void throwingEnricher_doesNotPreventPersist() {
        final TestEntry entry = buildEntry();

        // Must not throw even though ThrowingEnricher is registered
        repo.save(entry);

        assertThat(repo.findEntryById(entry.id)).isPresent();
    }

    @Test
    @Transactional
    void allEnrichersRun_despiteFailingEnricher() {
        final TestEntry entry = buildEntry();

        repo.save(entry);

        // CountingEnricher must have run (pipeline did not short-circuit on ThrowingEnricher)
        assertThat(CountingEnricher.count.get()).isGreaterThanOrEqualTo(1);
    }

    @Test
    @Transactional
    void traceId_stillPopulated_despiteThrowingEnricher() {
        // TraceIdEnricher must still run even when ThrowingEnricher is registered.
        // No active OTel span in this test, so traceId stays null — but the key point
        // is no exception propagates.
        final TestEntry entry = buildEntry();

        repo.save(entry);

        // Entry was saved — no exception means the pipeline completed gracefully
        assertThat(repo.findEntryById(entry.id)).isPresent();
    }

    private static TestEntry buildEntry() {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "pipeline-test-actor";
        e.actorType = ActorType.AGENT;
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 2: Add the `enricher-pipeline-test` profile datasource to `application.properties`**

Open `runtime/src/test/resources/application.properties` and add:

```properties
%enricher-pipeline-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:enricherpipelinetestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Step 3: Run to confirm failure** (ThrowingEnricher currently causes uncaught exception since old `LedgerTraceListener` didn't have try/catch — but we already refactored it, so these should actually PASS now)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LedgerEnricherPipelineIT -pl runtime
```

Expected: `Tests run: 3, Failures: 0`

- [ ] **Step 4: Run the full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerEnricherPipelineIT.java \
        runtime/src/test/resources/application.properties
git commit -m "test(enricher): pipeline non-fatal and multi-enricher integration tests

Refs #67"
```

---

### Task 6: Write ADR 0005 and update documentation

**Files:**
- Create: `adr/0005-ledger-entry-enricher-spi.md`
- Modify: `docs/DESIGN.md` (lines ~433 and ~462)
- Modify: `CLAUDE.md` (project structure table, LedgerTraceListener entry)

- [ ] **Step 1: Write ADR 0005**

```markdown
# ADR 0005 — LedgerEntry Enricher SPI

**Date:** 2026-04-28
**Status:** Accepted
**Refs:** #67

## Context

`LedgerTraceListener` auto-populated `traceId` via `@PrePersist`. Issue #59 was about
to add a second `@PrePersist` mechanism for provenance capture. Two separate listeners
for the same lifecycle concern would diverge, be applied inconsistently, and confuse
future contributors.

## Decision

Replace the monolithic `@PrePersist` body with a pluggable `LedgerEntryEnricher` SPI.
`LedgerTraceListener` becomes a non-fatal pipeline runner that iterates
`Instance<LedgerEntryEnricher>`. Any module can contribute enrichers by implementing
the interface as a CDI bean — no registration step required.

## Consequences

- `LedgerTraceListener` is now a thin pipeline runner; all enrichment logic lives in
  enricher beans.
- Enricher failures are logged and swallowed — the persist is never blocked.
- Enricher ordering is CDI-discovery order (unspecified). Enrichers must not depend on
  execution order.
- `#59` (ProvenanceSupplement) becomes a `ProvenanceCaptureEnricher` — one mechanism
  for all field auto-population at persist time.
```

- [ ] **Step 2: Update DESIGN.md**

Find the two mentions of `LedgerTraceListener` in the "Done" / capability table sections (around lines 433 and 462) and update them:

**Line ~433** — replace:
```
| OTel trace ID auto-wiring | ✅ Done — `LedgerTraceListener` auto-populates `traceId` (formerly `correlationId`) from the active OTel span at persist time. Closes #30, #31. |
```
with:
```
| OTel trace ID auto-wiring | ✅ Done — `TraceIdEnricher` auto-populates `traceId` from the active OTel span via the `LedgerEntryEnricher` pipeline (`LedgerTraceListener`). Closes #30, #31, #67. |
```

**Line ~462** — replace:
```
**OTel trace ID auto-wiring** — ✅ Done. `correlationId` renamed to `traceId`. `LedgerTraceListener` auto-populates `traceId` from the active OTel span at persist time. Closed #30, #31.
```
with:
```
**OTel trace ID auto-wiring** — ✅ Done. `LedgerTraceListener` runs the `LedgerEntryEnricher` pipeline at `@PrePersist`. `TraceIdEnricher` populates `traceId` from the active OTel span. New enrichers register by implementing `LedgerEntryEnricher` as a CDI bean — used by #59 (ProvenanceCaptureEnricher). Closed #30, #31, #67.
```

- [ ] **Step 3: Update CLAUDE.md project structure table**

Find the `LedgerTraceListener` entry and update it:

Replace:
```
│       ├── service/
│       │   ├── LedgerTraceListener.java        — JPA @EntityListeners hook: auto-populates traceId from OTel span
```
with:
```
│       ├── service/
│       │   ├── LedgerEntryEnricher.java         — SPI: pluggable @PrePersist enrichment pipeline
│       │   ├── TraceIdEnricher.java             — auto-populates traceId from active OTel span
│       │   ├── LedgerTraceListener.java         — @EntityListeners runner: iterates LedgerEntryEnricher pipeline, non-fatal
```

- [ ] **Step 4: Verify native image compatibility**

Quarkus's ArC CDI container processes `@ApplicationScoped` beans and `Instance<T>` injection at build time, so no explicit reflection config is normally needed. Verify by checking `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java` — if it already registers ledger service classes for reflection, add `LedgerEntryEnricher` and `TraceIdEnricher` to the same registration. If no special registration exists (ArC handles it), no action is needed. Document the outcome in ADR 0005 under a "Native image" note.

- [ ] **Step 5: Commit**

```bash
git add adr/0005-ledger-entry-enricher-spi.md docs/DESIGN.md CLAUDE.md
git commit -m "docs(enricher): ADR 0005, update DESIGN.md and CLAUDE.md for enricher pipeline

Closes #67"
```

---

## Part 2: #68 — ActorTrustScore Discriminator Model

### File Map

| Action | File | Purpose |
|---|---|---|
| Modify | `runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql` | Rewrite in place — add UUID PK, score_type, scope_key, NULLS NOT DISTINCT |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java` | Add ScoreType enum, UUID id, scoreType, scopeKey fields + named queries |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java` | Update upsert signature; add findByActorIdAndTypeAndKey, findByActorIdAndScoreType |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java` | Implement updated and new query methods |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` | Pass GLOBAL/null to upsert |
| Create | `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java` | Integration tests for all repository methods |
| Create | `adr/0006-actor-trust-score-discriminator-model.md` | ADR for the discriminator design |
| Modify | `docs/DESIGN.md` | Update trust score model description |
| Modify | `CLAUDE.md` | Update project structure table for ActorTrustScore |

---

### Task 7: Rewrite the V1001 migration

**Files:**
- Modify: `runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql`

Per CLAUDE.md: no production instances exist — rewrite in place, no incremental migration needed.

Key decisions:
- Surrogate `UUID id` as PK (actor_id is no longer unique by itself once score_type is added)
- `UNIQUE NULLS NOT DISTINCT (actor_id, score_type, scope_key)` — H2 2.4.240 and PostgreSQL 15+ both support this; two GLOBAL rows for the same actor correctly violate the constraint even though scope_key is NULL
- `scope_key` is nullable — NULL semantically means "no scope" (GLOBAL rows)
- `score_type DEFAULT 'GLOBAL'` — safe fallback

- [ ] **Step 1: Rewrite V1001**

```sql
-- CaseHub Ledger — actor trust score table (V1001)
-- Compatible with H2 2.4.240+ (dev/test) and PostgreSQL 15+ (production)
--
-- actor_trust_score: nightly-computed trust scores per actor and score type.
--
-- score_type:  GLOBAL     — one row per actor; classic Bayesian Beta score across all decisions
--              CAPABILITY — one row per (actor, capability tag); scoped Beta score (#61)
--              DIMENSION  — one row per (actor, trust dimension); e.g. thoroughness (#62)
--
-- scope_key:   NULL for GLOBAL rows; capability tag string for CAPABILITY; dimension name for DIMENSION.
--
-- Uniqueness: UNIQUE NULLS NOT DISTINCT (actor_id, score_type, scope_key)
--   NULLs are treated as equal for this constraint (PostgreSQL 15+ / H2 2.4.240+), so
--   two GLOBAL rows for the same actor correctly produce a constraint violation.
--
-- trust_score:        Bayesian Beta direct score: alpha_value / (alpha_value + beta_value)
-- global_trust_score: EigenTrust eigenvector component; values sum to ≤ 1.0 across all actors.
--                     Zero when EigenTrust is disabled or not yet computed (GLOBAL rows only).

CREATE TABLE actor_trust_score (
    id                   UUID             NOT NULL,
    actor_id             VARCHAR(255)     NOT NULL,
    score_type           VARCHAR(20)      NOT NULL DEFAULT 'GLOBAL',
    scope_key            VARCHAR(255),
    actor_type           VARCHAR(20),
    trust_score          DOUBLE PRECISION NOT NULL,
    global_trust_score   DOUBLE PRECISION NOT NULL DEFAULT 0.0,
    alpha_value          DOUBLE PRECISION NOT NULL,
    beta_value           DOUBLE PRECISION NOT NULL,
    decision_count       INT              NOT NULL,
    overturned_count     INT              NOT NULL,
    attestation_positive INT              NOT NULL,
    attestation_negative INT              NOT NULL,
    last_computed_at     TIMESTAMP,
    CONSTRAINT pk_actor_trust_score PRIMARY KEY (id),
    CONSTRAINT uq_actor_trust_score_key UNIQUE NULLS NOT DISTINCT (actor_id, score_type, scope_key)
);
```

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1001__actor_trust_score.sql
git commit -m "refactor(trust): rewrite V1001 — UUID PK, score_type/scope_key discriminator, NULLS NOT DISTINCT

Refs #68"
```

---

### Task 8: Write failing tests for backward-compat repository behaviour

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java`

Write the tests first. They will fail because `ActorTrustScore` still has `actorId` as `@Id` and no `ScoreType`.

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration tests for {@link ActorTrustScoreRepository} — covers all score types
 * and verifies backward compatibility of the GLOBAL score path.
 */
@QuarkusTest
@TestProfile(ActorTrustScoreRepositoryIT.RepoTestProfile.class)
class ActorTrustScoreRepositoryIT {

    public static class RepoTestProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "trust-repo-test";
        }
    }

    @Inject
    ActorTrustScoreRepository repo;

    // ── Backward compat: findByActorId still returns the GLOBAL score ─────────

    @Test
    @Transactional
    void findByActorId_returnsGlobalScore() {
        final String actorId = "actor-global-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                0.75, 5, 1, 2.0, 1.0, 4, 1, Instant.now());

        final var result = repo.findByActorId(actorId);

        assertThat(result).isPresent();
        assertThat(result.get().trustScore).isEqualTo(0.75);
        assertThat(result.get().scoreType).isEqualTo(ScoreType.GLOBAL);
        assertThat(result.get().scopeKey).isNull();
    }

    @Test
    @Transactional
    void findByActorId_returnsEmpty_whenOnlyCapabilityRowExists() {
        final String actorId = "actor-cap-only-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.85, 3, 0, 2.5, 1.0, 3, 0, Instant.now());

        // findByActorId is scoped to GLOBAL — must not return the CAPABILITY row
        assertThat(repo.findByActorId(actorId)).isEmpty();
    }

    // ── New: upsert is idempotent — second upsert updates, not inserts ────────

    @Test
    @Transactional
    void upsert_global_isIdempotent() {
        final String actorId = "actor-idem-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                0.5, 1, 0, 1.5, 1.5, 1, 0, Instant.now());
        repo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                0.8, 10, 2, 3.0, 1.0, 9, 1, Instant.now());

        final var all = repo.findAll();
        final long count = all.stream()
                .filter(s -> s.actorId.equals(actorId) && s.scoreType == ScoreType.GLOBAL)
                .count();
        assertThat(count).isEqualTo(1);
        assertThat(repo.findByActorId(actorId).get().trustScore).isEqualTo(0.8);
    }

    // ── New: scoped score queries ─────────────────────────────────────────────

    @Test
    @Transactional
    void findByActorIdAndTypeAndKey_returnsCapabilityScore() {
        final String actorId = "actor-scoped-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.85, 5, 0, 3.0, 1.0, 5, 0, Instant.now());

        final var result = repo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "security-review");

        assertThat(result).isPresent();
        assertThat(result.get().trustScore).isEqualTo(0.85);
        assertThat(result.get().scopeKey).isEqualTo("security-review");
    }

    @Test
    @Transactional
    void findByActorIdAndTypeAndKey_returnsEmpty_whenKeyDiffers() {
        final String actorId = "actor-wrongkey-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.85, 5, 0, 3.0, 1.0, 5, 0, Instant.now());

        assertThat(repo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "architecture-review"))
                .isEmpty();
    }

    @Test
    @Transactional
    void findByActorIdAndScoreType_returnsAllCapabilityRows() {
        final String actorId = "actor-multi-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.85, 5, 0, 3.0, 1.0, 5, 0, Instant.now());
        repo.upsert(actorId, ScoreType.CAPABILITY, "architecture-review", ActorType.AGENT,
                0.60, 3, 1, 2.0, 1.5, 2, 1, Instant.now());

        final var results = repo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);

        assertThat(results).hasSize(2);
        assertThat(results).extracting(s -> s.scopeKey)
                .containsExactlyInAnyOrder("security-review", "architecture-review");
    }

    @Test
    @Transactional
    void upsert_capability_isIdempotent() {
        final String actorId = "actor-cap-idem-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.5, 1, 0, 1.5, 1.5, 1, 0, Instant.now());
        repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", ActorType.AGENT,
                0.9, 10, 0, 5.0, 1.0, 10, 0, Instant.now());

        final var results = repo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).trustScore).isEqualTo(0.9);
    }

    // ── updateGlobalTrustScore still works ────────────────────────────────────

    @Test
    @Transactional
    void updateGlobalTrustScore_updatesExistingGlobalRow() {
        final String actorId = "actor-eigentrust-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                0.75, 5, 0, 2.0, 1.0, 5, 0, Instant.now());

        repo.updateGlobalTrustScore(actorId, 0.42);

        assertThat(repo.findByActorId(actorId).get().globalTrustScore).isEqualTo(0.42);
    }
}
```

- [ ] **Step 2: Add the `trust-repo-test` profile datasource to `application.properties`**

```properties
%trust-repo-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:trustrepotest;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Step 3: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ActorTrustScoreRepositoryIT -pl runtime
```

Expected: compilation or runtime failure — `ScoreType` does not exist; `upsert` signature mismatch.

---

### Task 9: Update ActorTrustScore entity

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`

- [ ] **Step 1: Rewrite the entity**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

/**
 * Bayesian Beta trust score for a decision-making actor, scoped by score type.
 *
 * <p>One row per {@code (actor_id, score_type, scope_key)} triple:
 * <ul>
 *   <li>{@code GLOBAL} — one row per actor; classic score across all decisions. {@code scope_key} is null.</li>
 *   <li>{@code CAPABILITY} — one row per (actor, capability tag); requires #61.</li>
 *   <li>{@code DIMENSION} — one row per (actor, trust dimension); requires #62.</li>
 * </ul>
 *
 * <p>Plain {@code @Entity} — queries via {@code @NamedQuery} + EntityManager.
 */
@Entity
@Table(
        name = "actor_trust_score",
        uniqueConstraints = @UniqueConstraint(
                name = "uq_actor_trust_score_key",
                columnNames = {"actor_id", "score_type", "scope_key"}))
@NamedQuery(name = "ActorTrustScore.findAll",
        query = "SELECT s FROM ActorTrustScore s")
@NamedQuery(name = "ActorTrustScore.findGlobalByActorId",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.scopeKey IS NULL")
@NamedQuery(name = "ActorTrustScore.findByActorIdAndScoreType",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType")
@NamedQuery(name = "ActorTrustScore.findByActorIdAndTypeAndKey",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.scopeKey = :scopeKey")
public class ActorTrustScore {

    /** Score type discriminator — determines what scope_key means. */
    public enum ScoreType {
        /** Classic cross-decision score. scope_key is null. */
        GLOBAL,
        /** Capability-scoped score. scope_key is the capability tag (e.g. "security-review"). Requires #61. */
        CAPABILITY,
        /** Dimension-scoped score. scope_key is the dimension name (e.g. "thoroughness"). Requires #62. */
        DIMENSION
    }

    @Id
    public UUID id;

    @Column(name = "actor_id", nullable = false)
    public String actorId;

    @Enumerated(EnumType.STRING)
    @Column(name = "score_type", nullable = false)
    public ScoreType scoreType = ScoreType.GLOBAL;

    /** Null for GLOBAL rows; capability tag for CAPABILITY; dimension name for DIMENSION. */
    @Column(name = "scope_key")
    public String scopeKey;

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

    /**
     * EigenTrust global trust share in [0.0, 1.0]; values sum to ≤ 1.0 across all actors.
     * Only meaningful on GLOBAL rows. Zero when EigenTrust is disabled or not yet computed.
     */
    @Column(name = "global_trust_score")
    public double globalTrustScore;
}
```

---

### Task 10: Update ActorTrustScoreRepository SPI

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java`

- [ ] **Step 1: Rewrite the interface**

```java
package io.casehub.ledger.runtime.repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorType;

/** SPI for persisting and querying {@link ActorTrustScore} records. */
public interface ActorTrustScoreRepository {

    /**
     * Find the GLOBAL trust score for an actor, or empty if none computed yet.
     * Backward-compatible shorthand for {@code findByActorIdAndTypeAndKey(actorId, GLOBAL, null)}.
     */
    Optional<ActorTrustScore> findByActorId(String actorId);

    /**
     * Find a scoped trust score for an actor by type and scope key.
     * For GLOBAL scores, use {@link #findByActorId(String)} or pass {@code scopeKey = null}.
     *
     * @param scopeKey null for GLOBAL; capability tag for CAPABILITY; dimension name for DIMENSION
     */
    Optional<ActorTrustScore> findByActorIdAndTypeAndKey(String actorId, ScoreType scoreType, String scopeKey);

    /**
     * Return all trust scores for an actor of a given type.
     * For GLOBAL: returns 0 or 1 result. For CAPABILITY/DIMENSION: returns all scoped rows.
     */
    List<ActorTrustScore> findByActorIdAndScoreType(String actorId, ScoreType scoreType);

    /**
     * Upsert (insert or update) a trust score for the given actor and scope.
     *
     * @param scoreType the score type (GLOBAL, CAPABILITY, DIMENSION)
     * @param scopeKey  null for GLOBAL; capability tag or dimension name otherwise
     */
    void upsert(String actorId, ScoreType scoreType, String scopeKey,
            ActorType actorType, double trustScore,
            int decisionCount, int overturnedCount, double alpha, double beta,
            int attestationPositive, int attestationNegative,
            Instant lastComputedAt);

    /**
     * Update the EigenTrust global trust score for an actor's GLOBAL row.
     */
    void updateGlobalTrustScore(String actorId, double globalTrustScore);

    /**
     * Return all computed trust scores across all actors and score types.
     */
    List<ActorTrustScore> findAll();
}
```

---

### Task 11: Implement the updated JPA repository

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`

- [ ] **Step 1: Rewrite the implementation**

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;

/**
 * JPA / EntityManager implementation of {@link ActorTrustScoreRepository}.
 *
 * <p>Upsert is a find-then-update to remain compatible with H2 and PostgreSQL without
 * database-specific SQL. The unique constraint (actor_id, score_type, scope_key) with
 * NULLS NOT DISTINCT prevents duplicate GLOBAL rows at the database level.
 */
@ApplicationScoped
public class JpaActorTrustScoreRepository implements ActorTrustScoreRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return em.createNamedQuery("ActorTrustScore.findGlobalByActorId", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.GLOBAL)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<ActorTrustScore> findByActorIdAndTypeAndKey(
            final String actorId, final ScoreType scoreType, final String scopeKey) {
        if (scopeKey == null) {
            return em.createNamedQuery("ActorTrustScore.findGlobalByActorId", ActorTrustScore.class)
                    .setParameter("actorId", actorId)
                    .setParameter("scoreType", scoreType)
                    .getResultStream()
                    .findFirst();
        }
        return em.createNamedQuery("ActorTrustScore.findByActorIdAndTypeAndKey", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", scoreType)
                .setParameter("scopeKey", scopeKey)
                .getResultStream()
                .findFirst();
    }

    @Override
    public List<ActorTrustScore> findByActorIdAndScoreType(
            final String actorId, final ScoreType scoreType) {
        return em.createNamedQuery("ActorTrustScore.findByActorIdAndScoreType", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", scoreType)
                .getResultList();
    }

    @Override
    @Transactional
    public void upsert(final String actorId, final ScoreType scoreType, final String scopeKey,
            final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount,
            final double alpha, final double beta,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {

        ActorTrustScore score = findByActorIdAndTypeAndKey(actorId, scoreType, scopeKey).orElse(null);
        if (score == null) {
            score = new ActorTrustScore();
            score.id = UUID.randomUUID();
            score.actorId = actorId;
            score.scoreType = scoreType;
            score.scopeKey = scopeKey;
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

    @Override
    @Transactional
    public void updateGlobalTrustScore(final String actorId, final double globalTrustScore) {
        findByActorId(actorId).ifPresent(score -> {
            score.globalTrustScore = globalTrustScore;
            em.merge(score);
        });
    }

    @Override
    public List<ActorTrustScore> findAll() {
        return em.createNamedQuery("ActorTrustScore.findAll", ActorTrustScore.class)
                .getResultList();
    }
}
```

---

### Task 12: Update TrustScoreJob to pass GLOBAL/null

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Update the upsert call**

In `TrustScoreJob.runComputation()`, find the `trustRepo.upsert(...)` call and update the signature:

Replace:
```java
trustRepo.upsert(actorId, actorType, score.trustScore(),
        score.decisionCount(), score.overturnedCount(),
        score.alpha(), score.beta(),
        score.attestationPositive(), score.attestationNegative(), now);
```
With:
```java
trustRepo.upsert(actorId, ActorTrustScore.ScoreType.GLOBAL, null,
        actorType, score.trustScore(),
        score.decisionCount(), score.overturnedCount(),
        score.alpha(), score.beta(),
        score.attestationPositive(), score.attestationNegative(), now);
```

Also add the import at the top of the file:
```java
import io.casehub.ledger.runtime.model.ActorTrustScore;
```

- [ ] **Step 2: Verify previousSnapshot keying is still safe**

The `previousSnapshot` map keys by `s.actorId`. After #68, `findAll()` returns all score types. For now (with only GLOBAL rows in existence), this is identical behaviour. When #61/#62 ship CAPABILITY/DIMENSION rows, the map will have collisions and `TrustScoreRoutingPublisher` will need updating — open a comment on #61 noting this. For this PR: verify the line reads:
```java
.collect(Collectors.toMap(s -> s.actorId, s -> s));
```
and leave a `// TODO #61: restrict to GLOBAL rows when CAPABILITY rows are added` comment above it.

---

### Task 13: Run all tests

- [ ] **Step 1: Run the new repository IT and the full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass including the new `ActorTrustScoreRepositoryIT` (8 tests) and unchanged `TrustScoreIT` (5 tests).

- [ ] **Step 2: Install the SNAPSHOT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
```

- [ ] **Step 3: Commit everything**

```bash
git add \
  runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
  runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java \
  runtime/src/test/resources/application.properties
git commit -m "refactor(trust): ActorTrustScore discriminator model — ScoreType, scope_key, UUID PK

All existing GLOBAL score behaviour unchanged. New query methods ready for
capability-scoped (#61) and multi-dimensional (#62) trust scores.
UNIQUE NULLS NOT DISTINCT constraint enforced at DB level (H2 2.4.240 / PG 15+).

Refs #68"
```

---

### Task 14: Write ADR 0006 and update documentation

**Files:**
- Create: `adr/0006-actor-trust-score-discriminator-model.md`
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Write ADR 0006**

```markdown
# ADR 0006 — ActorTrustScore Discriminator Model

**Date:** 2026-04-28
**Status:** Accepted
**Refs:** #68

## Context

Issues #61 (capability-scoped trust) and #62 (multi-dimensional trust) were both about
to add nullable columns to `actor_trust_score`. Two separate nullable scope keys on the
same table is messy, hard to query, and closed to extension. A unified discriminator
model was designed before either feature begins.

## Decision

Replace the single-row-per-actor model with a `(actor_id, score_type, scope_key)` discriminator:

- Surrogate UUID `id` as primary key (actor_id is no longer globally unique)
- `score_type` enum: GLOBAL | CAPABILITY | DIMENSION
- `scope_key` nullable: NULL for GLOBAL, tag string for CAPABILITY, dimension name for DIMENSION
- `UNIQUE NULLS NOT DISTINCT (actor_id, score_type, scope_key)` — enforces that two GLOBAL rows
  for the same actor are a constraint violation, even though scope_key is NULL

`NULLS NOT DISTINCT` is supported in PostgreSQL 15+ and H2 2.4.240+ (the version pulled by
Quarkus 3.32.2). No sentinel value (`''`) is needed.

## Rejected alternative: empty string sentinel for GLOBAL scope_key

Using `scope_key = ''` for GLOBAL avoids the NULL constraint issue but is semantically
misleading (`''` does not mean "no scope") and carries a maintenance burden. Checking library
versions revealed `NULLS NOT DISTINCT` is available in all target databases — no workaround needed.

## Consequences

- All existing `findByActorId` callers continue to work — the method returns the GLOBAL row.
- `#61` and `#62` add computation logic only — the schema and query infrastructure are already in place.
- Adding a new score type (e.g. TEMPORAL) requires only a new `ScoreType` enum value — no schema change.
- EigenTrust (`globalTrustScore`) is only meaningful on GLOBAL rows; this is documented in Javadoc.
```

- [ ] **Step 2: Update DESIGN.md trust score section**

Find the paragraph around line 448 that reads:
```
`ActorTrustScore` stores `trust_score`, `alpha_value`, `beta_value`, diagnostic
counters, and `global_trust_score` (EigenTrust). `TrustScoreJob` runs nightly when enabled.
```

Replace with:
```
`ActorTrustScore` uses a discriminator model keyed by `(actor_id, score_type, scope_key)`.
`score_type` is `GLOBAL` (classic cross-decision Beta score), `CAPABILITY` (scoped to a
capability tag — wired by #61), or `DIMENSION` (scoped to a trust dimension — wired by #62).
`scope_key` is null for GLOBAL rows; the unique constraint uses `NULLS NOT DISTINCT` to
enforce one GLOBAL row per actor. `TrustScoreJob` writes GLOBAL rows; capability and dimension
computation is added by #61 and #62 respectively.
```

- [ ] **Step 3: Update CLAUDE.md project structure table**

Find the `ActorTrustScore.java` entry and update the comment:

Replace:
```
│   ├── ActorTrustScore.java             — nightly-computed reputation entity
```
With:
```
│   ├── ActorTrustScore.java             — trust score entity; discriminator model (GLOBAL|CAPABILITY|DIMENSION) × scope_key
```

- [ ] **Step 4: Commit**

```bash
git add adr/0006-actor-trust-score-discriminator-model.md docs/DESIGN.md CLAUDE.md
git commit -m "docs(trust): ADR 0006, update DESIGN.md and CLAUDE.md for discriminator model

Closes #68"
```

---

## Final Verification

- [ ] **Run the complete test suite one last time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: all tests pass. Count should be ≥ 223 (212 baseline + 3 enricher pipeline + 8 repository IT).

- [ ] **Reinstall the SNAPSHOT for consumers**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
```

- [ ] **Step 6 consolidation checks**

After both issues close, run these greps across all consumer repos:

```bash
# #67: any other @EntityListeners on ledger-related entities?
grep -r "@EntityListeners.*[Ll]edger" \
  ~/claude/casehub/engine ~/claude/casehub-work ~/claude/casehub-qhorus ~/claude/claudony

# #68: any code querying ActorTrustScore directly rather than via repository?
grep -r "ActorTrustScore" \
  ~/claude/casehub/engine ~/claude/casehub-work ~/claude/casehub-qhorus ~/claude/claudony
```

Open tracked issues for any consolidation work found. Both issues' Step 6 checks are satisfied by this step.
