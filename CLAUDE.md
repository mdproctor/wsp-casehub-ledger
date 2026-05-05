# ledger Workspace

**Project repo:** /Users/mdproctor/claude/casehub/ledger
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/ledger` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `epic` at branch start)

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | workspace   | |
| blog       | workspace   | |
| design     | workspace   | |
| snapshots  | workspace   | |
| specs      | workspace   | |
| handover   | workspace   | |

---

# CaseHub Ledger — Claude Code Project Guide

## Platform Context

This repo is one component of the casehubio multi-repo platform. **Before implementing anything — any feature, SPI, data model, or abstraction — run the Platform Coherence Protocol.**

The protocol asks: Does this already exist elsewhere? Is this the right repo for it? Does this create a consolidation opportunity? Is this consistent with how the platform handles the same concern in other repos?

**Platform architecture (fetch before any implementation decision):**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md
```

**This repo's deep-dive:**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-ledger.md
```

**Other repo deep-dives** (fetch the relevant ones when your implementation touches their domain):
- casehub-work: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-work.md`
- casehub-qhorus: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-qhorus.md`
- casehub-engine: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-engine.md`
- claudony: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/claudony.md`
- casehub-connectors: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-connectors.md`

---

## Project Type

type: java

**Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, GraalVM 25 (native image)

---

## What This Project Is

`casehub-ledger` is a CaseHub extension providing a domain-agnostic immutable
audit ledger for any Quarkus application. Any Quarkus app adds `io.casehub:casehub-ledger`
as a dependency and immediately gets:

- **Immutable append-only audit log** (`LedgerEntry` base entity with JPA JOINED inheritance)
- **Merkle Mountain Range tamper evidence** (RFC 9162 stored frontier — O(log N) inclusion proofs, Ed25519 signed checkpoints)
- **Peer attestation** (`LedgerAttestation` — verdicts, confidence scores)
- **EigenTrust reputation** (`TrustScoreComputer` — nightly batch, exponential decay weighting)
- **Provenance tracking** (`sourceEntityId / sourceEntityType / sourceEntitySystem`)
- **Decision context snapshots** (GDPR Article 22 / EU AI Act Article 12 compliance)

### Domain-Specific Subclasses

Domain logic is NOT in this extension — it lives in consumers via JPA JOINED subclasses:

| Consumer | Subclass | Subclass table | subject_id maps to |
|---|---|---|---|
| `casehub-work` | `WorkItemLedgerEntry` | `work_item_ledger_entry` | WorkItem UUID |
| `casehub-qhorus` | `MessageLedgerEntry` | `message_ledger_entry` | Channel UUID |

Each consumer defines its own subclass and its own Flyway migration for the subclass table.
The base tables (`ledger_entry`, `ledger_attestation`, `actor_trust_score`) are defined here
in V1000–V1004 and always present when `casehub-ledger` is on the classpath.

**Design documentation:** `docs/DESIGN.md` covers entity model, architecture, SPI contracts, and configuration. `docs/DESIGN-capabilities.md` covers Merkle MMR, PROV-DM export, agent identity model, and agent mesh topology.

---

## Maven Coordinates

| Element | Value |
|---|---|
| GitHub repo | `casehubio/ledger` |
| groupId | `io.casehub` |
| Parent artifactId | `casehub-ledger-parent` |
| Runtime artifactId | `casehub-ledger` |
| Deployment artifactId | `casehub-ledger-deployment` |
| Root Java package | `io.casehub.ledger.runtime` |
| Deployment subpackage | `io.casehub.ledger.deployment` |
| Config prefix | `casehub.ledger` |
| Feature name | `ledger` |

---

## Key Design Decisions

**`subject_id` — the generic aggregate identifier**
All queries, sequences, and hash chains are scoped per `subject_id`. This field replaces
the domain-specific `work_item_id` that was in the original Tarkus ledger. Consumers set
`subjectId` to their own aggregate UUID (WorkItem UUID, Channel UUID, etc.).

**JPA JOINED inheritance**
`LedgerEntry` is abstract with `@Inheritance(strategy = JOINED)`. Hibernate joins to all
registered subclass tables on query. `LedgerAttestation` holds a FK to the base table —
attestations work regardless of which subclass produced the entry.

**Merkle leaf hash canonical form (core fields only)**
`subjectId|seqNum|entryType|actorId|actorRole|occurredAt`
Domain-specific subclass fields and supplement fields are excluded — canonical form stays
domain-agnostic. The leaf hash is `SHA-256(0x00 | canonicalBytes)` per RFC 9162.
The Merkle Mountain Range (stored frontier) replaces the old linear chain.

**`traceId` and `causedByEntryId` are core fields**
Both OTel trace linking and causal relationships are structural — present on every entry where
relevant. They live on `LedgerEntry` directly (not in supplements). `traceId` is auto-populated
from the active OTel span at persist time via the `LedgerEntryEnricher` pipeline (`LedgerTraceListener`). `findCausedBy(UUID entryId)`
traverses causal chains one hop at a time. The test for core vs supplement: is the field
relevant to every consumer, every entry, every time? If yes → core. If no → supplement.

**All entities are plain `@Entity` — no Panache active-record base**
No entity in the runtime module extends `PanacheEntityBase`. This allows reactive
subclassing by consumers (e.g. Qhorus's `MessageLedgerEntry`) and removes the
forced `quarkus-hibernate-orm-panache` dep. Repositories use `EntityManager` + JPQL.
Queries are declared as `@NamedQuery` on entity classes — Hibernate validates them at
startup, so typos fail at boot not at query time.
`LedgerEntryRepository.findById(UUID)` was renamed to `findEntryById(UUID)` to avoid
a Java return-type conflict with `PanacheRepositoryBase.findById()`.

**REST endpoints are domain-specific**
`casehub-ledger` provides model, SPI, services, and JPA implementations only. Tarkus and
Qhorus each define their own REST/MCP endpoints on top.

**`actorId` format for LLM agents**
LLM agents are stateless; use versioned persona names so trust accumulates correctly
across sessions: `"{model-family}:{persona}@{major}"` — e.g. `"claude:tarkus-reviewer@v1"`.
Major version bump resets the trust baseline; tuning/bug-fix does not. See ADR 0004 and
`docs/DESIGN-capabilities.md` (Agent Identity Model) for concrete bump criteria and the no-inheritance
rationale.

---

## Project Structure

```
casehub-ledger/  (local folder: ~/claude/casehub/ledger)
├── runtime/
│   └── src/main/java/io/casehub/ledger/runtime/
│       ├── config/LedgerConfig.java         — @ConfigMapping(prefix = "casehub.ledger")
│       ├── model/
│       │   ├── LedgerEntry.java             — abstract base entity (JOINED inheritance)
│       │   ├── LedgerAttestation.java       — peer attestation entity
│       │   ├── ActorTrustScore.java         — trust score entity; discriminator model (GLOBAL|CAPABILITY|DIMENSION) × scope_key
│       │   ├── LedgerMerkleFrontier.java    — Merkle frontier node entity (log₂(N) rows per subject)
│       │   ├── LedgerEntryArchiveRecord.java — archive snapshot record for retention-deleted entries (V1003)
│       │   ├── LedgerEntryType.java         — COMMAND | EVENT | ATTESTATION (api module)
│       │   ├── ActorType.java               — HUMAN | AGENT | SYSTEM (api module)
│       │   ├── AttestationVerdict.java      — SOUND | FLAGGED | ENDORSED | CHALLENGED (api module)
│       │   ├── CapabilityTag.java           — sentinel constants: GLOBAL = "*" for cross-capability attestations (api module)
│       │   ├── ActorIdentity.java           — token↔identity mapping for pseudonymisation
│       │   └── supplement/
│       │       ├── LedgerSupplement.java        — abstract base (JOINED inheritance)
│       │       ├── ComplianceSupplement.java    — GDPR Art.22, governance fields
│       │       ├── ProvenanceSupplement.java    — workflow source entity; agentConfigHash for LLM config drift detection
│       │       └── LedgerSupplementSerializer.java — JSON serialiser for supplementJson
│       ├── repository/
│       │   ├── LedgerEntryRepository.java        — blocking SPI (uses subjectId); findById → findEntryById
│       │   ├── ReactiveLedgerEntryRepository.java — reactive SPI (Uni<T> return types)
│       │   ├── ActorTrustScoreRepository.java     — SPI
│       │   └── jpa/                              — JPA implementations (EntityManager-based)
│       ├── service/
│       │   ├── LedgerEntryEnricher.java         — SPI: pluggable @PrePersist enrichment pipeline
│       │   ├── TraceIdEnricher.java             — auto-populates traceId from active OTel span
│       │   ├── OtelTraceIdProvider.java         — OTel span reader for TraceIdEnricher
│       │   ├── LedgerTraceListener.java         — @EntityListeners runner: iterates LedgerEntryEnricher pipeline, non-fatal
│       │   ├── LedgerMerkleTree.java            — Merkle Mountain Range algorithm (pure static)
│       │   ├── LedgerVerificationService.java   — treeRoot / inclusionProof / verify (CDI bean)
│       │   ├── LedgerMerklePublisher.java       — Ed25519 signed tlog-checkpoint (opt-in CDI bean)
│       │   ├── LedgerProvExportService.java      — W3C PROV-DM JSON-LD export (CDI bean)
│       │   ├── LedgerProvSerializer.java         — PROV-DM serialisation utility
│       │   ├── LedgerEntryArchiver.java          — archive record JSON serialisation for retention
│       │   ├── model/
│       │   │   ├── InclusionProof.java       — Merkle inclusion proof value type
│       │   │   └── ProofStep.java            — single sibling node in a proof path
│       │   ├── RetentionEligibilityChecker.java — pure utility: checks retention window eligibility per entry
│       │   ├── LedgerRetentionJob.java      — @Scheduled daily retention sweep (EU AI Act Art.12)
│       │   ├── DecayFunction.java           — SPI: attestation decay weight (ageInDays, verdict) → weight
│       │   ├── ExponentialDecayFunction.java — @DefaultBean: 2^(-age/halfLife) × valence multiplier (FLAGGED slower decay)
│       │   ├── TrustScoreComputer.java      — Bayesian Beta trust scoring (compute) + decay-weighted dimension average (computeDimensionScore); delegates decay to DecayFunction (pure Java)
│       │   ├── TrustGateService.java        — CDI bean: trust threshold enforcement (meetsThreshold, currentScore, dimensionScores, dimensionScore)
│       │   ├── GlobalScoreStrategy.java          — SPI: select attestations / derive global trust score (ADR 0008)
│       │   ├── AllAttestationsGlobalStrategy.java — @DefaultBean: all attestations → global Beta (Option B)
│       │   ├── ExplicitGlobalAttestationsStrategy.java — @Alternative: only "*" attestations (Option A)
│       │   ├── FrequencyWeightedGlobalStrategy.java — @Alternative: frequency-weighted from capability scores (Option C)
│       │   ├── AttestationAggregator.java   — CDI bean: collapses (entryId, capabilityTag) attestation groups into consensus verdict (WEIGHTED_MAJORITY | UNANIMOUS_REQUIRED | FIRST_ATTESTOR)
│       │   ├── EigenTrustComputer.java      — EigenTrust power iteration, transitive global trust scores (pure Java)
│       │   ├── TrustScoreJob.java           — @Scheduled nightly recomputation (capability pass → dimension pass → global pass)
│       │   ├── LedgerHealthJob.java         — @Scheduled gap detection + reconciliation (configurable interval, default 1h)
│       │   ├── LedgerReconciliationSource.java — SPI: consumers implement to compare domain entity counts vs ledger counts
│       │   ├── LedgerGapDetected.java       — CDI event fired on sequence gap or reconciliation mismatch
│       │   ├── GapType.java                 — SEQUENCE_GAP | RECONCILIATION_MISMATCH
│       │   ├── LedgerComplianceReportService.java — CDI bean: reportForActor / reportForSubject → ComplianceReport
│       │   ├── ComplianceReport.java        — value type: DecisionRecord list + Merkle anchor + format(ReportFormat)
│       │   ├── DecisionRecord.java          — single automated decision entry in a compliance report
│       │   ├── ReportFormat.java            — PLAIN_JSON | JSON_LD | CSV
│       │   ├── routing/
│       │   │   ├── TrustScoreRoutingPublisher.java — CDI event dispatch after trust score computation
│       │   │   ├── TrustScoreFullPayload.java      — all current scores (strategy: rebuild ranked list)
│       │   │   ├── TrustScoreDeltaPayload.java     — changed actors only (strategy: incremental cache)
│       │   │   ├── TrustScoreComputedAt.java       — lightweight notification (strategy: signal only)
│       │   │   └── TrustScoreDelta.java            — single actor score change value type
│       │   └── intercept/
│       │       ├── ProvenanceCapture.java           — CDI interceptor binding (@InterceptorBinding); attributes sourceEntityType, sourceEntitySystem
│       │       ├── ProvenanceCaptureInterceptor.java — CDI interceptor: pushes ProvenanceContext before proceed, pops in finally
│       │       ├── ProvenanceCaptureEnricher.java   — LedgerEntryEnricher: attaches ProvenanceSupplement from active context
│       │       ├── ProvenanceContext.java           — @ApplicationScoped ThreadLocal stack; supports nested @ProvenanceCapture scopes
│       │       └── SourceEntityId.java              — parameter annotation: marks the UUID to use as sourceEntityId
│       └── privacy/
│           ├── ActorIdentityProvider.java   — SPI: tokenise/resolve/erase actor identities
│           ├── DecisionContextSanitiser.java — SPI: sanitise decisionContext JSON before persist
│           ├── InternalActorIdentityProvider.java — built-in UUID token impl (config-gated)
│           ├── LedgerErasureService.java    — GDPR Art.17 erasure (CDI bean)
│           └── LedgerPrivacyProducer.java   — CDI producer for both SPIs (@DefaultBean)
│   └── src/main/resources/db/migration/
│       ├── V1000__ledger_base_schema.sql    — ledger_entry + ledger_attestation tables
│       ├── V1001__actor_trust_score.sql     — actor_trust_score discriminator model (UUID PK, score_type GLOBAL|CAPABILITY|DIMENSION, scope_key, NULLS NOT DISTINCT)
│       ├── V1002__ledger_supplement.sql     — supplement tables + drops moved columns
│       ├── V1003__ledger_entry_archive.sql  — ledger_entry_archive table
│       └── V1004__actor_identity.sql        — actor_identity pseudonymisation table
└── deployment/
    └── src/main/java/io/casehub/ledger/deployment/
        └── LedgerProcessor.java             — @BuildStep: FeatureBuildItem
```

---

## Build and Test

```bash
# Build all modules
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install

# Run all tests (all modules)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test

# Run tests for a specific module
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime   # QuarkusTest + IT
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api        # pure JUnit 5, no Quarkus runtime

# If you edited api/ and want to run only runtime tests, install api first:
# (mvn test -pl runtime resolves api from .m2 cache — source changes in api are invisible otherwise)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime

# Native image build (requires GraalVM)
JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home \
  mvn package -Pnative -DskipTests
```

**Use `mvn` not `./mvnw`** — maven wrapper not configured on this machine.

---

## Java and GraalVM on This Machine

```bash
# Java 26 (Oracle, system default) — use for dev and tests
JAVA_HOME=$(/usr/libexec/java_home -v 26)

# GraalVM 25 — use for native image builds only
JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home
```

---

## Ecosystem Context

```
casehub-ledger       (audit/provenance — this project)
    ↑         ↑
 casehub-work    casehub-qhorus    (each adds its own LedgerEntry subclass)
    ↑         ↑
          claudony
```

casehub-work and casehub-qhorus are siblings — neither depends on the other. Both depend on
`casehub-ledger`. Claudony composes them.

---

## Schema Convention

**No existing installations** — there are no deployed instances of `casehub-ledger` in production.
All schema changes go directly into the base migration files (V1000–V1004) or into a new base
migration file. Do NOT create incremental migration scripts to evolve the schema. Rewrite the
relevant migration file in place. Treat every schema change as a clean-slate design decision.

---

## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** casehubio/ledger
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding", "execute the plan", "let's build", or similar: check if an active issue or epic exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be implemented. If not, draft one and assess epic placement (issue-workflow Phase 2) before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm issue linkage and check for split candidates. This is a fallback — the issue should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done). If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm before proceeding — it must be a deliberate choice, not a default.

---

## Writing Style Guide

**The writing style guide at `~/claude-workspace/writing-styles/blog-technical.md` is mandatory for all blog and diary entries.** Load it in full before drafting. Complete the pre-draft voice classification (I / we / Claude-named) before generating any prose. Do not show a draft without verifying it against the style guide.

**Blog directory:** `blog/`

## Ecosystem Conventions

All casehubio projects align on these conventions:

**Quarkus version:** All projects use `3.32.2`. When bumping, bump all projects together.

**GitHub Packages — dependency resolution:** Add to `pom.xml` `<repositories>`:
```xml
<repository>
  <id>github</id>
  <url>https://maven.pkg.github.com/casehubio/*</url>
  <snapshots><enabled>true</enabled></snapshots>
</repository>
```
CI must use `server-id: github` + `GITHUB_TOKEN` in `actions/setup-java`.

**Cross-project SNAPSHOT versions:** `casehub-ledger` and `casehub-work` modules are `0.2-SNAPSHOT` resolved from GitHub Packages. Declare in `pom.xml` properties and `<dependencyManagement>` — no hardcoded versions in submodule poms.
