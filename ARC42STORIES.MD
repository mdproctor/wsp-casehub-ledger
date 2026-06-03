# casehub-ledger — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** Early foundation — no casehubio dependencies. Must publish before qhorus, eidos, engine, and all application-tier apps.
**Consumed by:** casehub-qhorus (MessageLedgerEntry), casehub-eidos (runtime), casehub-engine (optional ledger integration), all application-tier harness apps
**Depends on:** None — zero casehubio dependencies.

---

## §1 Introduction and Goals

### Description

`casehub-ledger` is the immutable tamper-evident audit and trust scoring layer. It provides a Merkle-chained ledger of all case events and agent decisions, trust score computation from outcome attestations, and GDPR ComplianceSupplement for right-to-erasure compliance.

Core capabilities: `LedgerEntry` (append-only, hash-chained audit record), `CaseLedgerEntry` (case state transition record with `causedByEntryId` for production incident traceability), `MessageLedgerEntry` (per-speech-act qhorus message record), `WorkerDecisionEntry` (agent outcome attestation — input to trust scoring), `TrustScoreJob` (Bayesian Beta trust score computation from `WorkerDecisionEntry` records), `ComplianceSupplement` (GDPR Art.17 erasure metadata), `MerkleChain` (append-only hash frontier).

Module structure: `api` (LedgerEntry types, SPIs), `runtime` (Merkle chain, trust scoring, JPA persistence), `deployment` (Quarkus extension), `persistence-memory` (zero-datasource in-memory SPIs for @QuarkusTest isolation).

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Tamper-evidence | Every LedgerEntry is hash-chained to its predecessor — no entry can be modified without breaking the chain |
| GDPR compliance | ComplianceSupplement enables per-entry right-to-erasure without breaking the Merkle chain |
| Trust accuracy | TrustScoreJob uses Bayesian Beta distribution — cold-start agents route by availability; quality improves as WorkerDecisionEntry attestations accumulate |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-ledger:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-ledger owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

### 2026-06-03 — Native image resource registration (LedgerProcessor)

`LedgerProcessor` produces `NativeImageResourcePatternsBuildItem` registering
`db/ledger/migration/*.sql` for inclusion in native image builds. Without this
`@BuildStep`, GraalVM's resource collection phase silently omits the SQL files —
classpath scanning that works at JVM runtime is replaced at native-image time by
a pre-registered list, and any resource not in that list is absent in the binary.
The deployment module is the correct place for this registration: it is the
build-time processor that knows which classpath paths the extension owns, and
`NativeImageResourcePatternsBuildItem` is a deployment-module build item.
Consumers do not need to add anything — the extension self-registers its own resources.
(Closes ledger#99. Protocol PP-20260528-flyway-ext-reg known gap now resolved.)

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `LedgerEntry` | Immutable, hash-chained audit record. Once written, never modified. |
| `CaseLedgerEntry` | Case state transition record — stores causedByEntryId to trace production incidents to specific review decisions. |
| `WorkerDecisionEntry` | Agent outcome attestation — the input to TrustScoreJob for capability-scoped trust score computation. |
| `TrustScoreJob` | Periodic job that reads WorkerDecisionEntry records and updates per-agent, per-capability Bayesian Beta trust scores. |
| `ComplianceSupplement` | GDPR metadata attached to a LedgerEntry — enables erasure of personal data without invalidating the chain. |
| `MerkleChain` | Append-only hash frontier — each entry's hash includes the previous entry's hash. |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
