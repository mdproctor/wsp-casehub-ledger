# Eidos — Specification

**Status:** Pre-implementation spec  
**Started:** 2026-05-22  
**Name:** Eidos  
**Strategy:** Build first. Engage A2A / open standard community when casehub is ready.  
**Research backing:** `agent-description-ontology.md`, `casehub-platform-vocabulary-validation.md`

---

## What Is Eidos?

Eidos is a new casehub platform repo that gives LLM agents a structured, discoverable, and generative identity.

Three things it enables:

1. **Description** — a structured `AgentDescriptor` capturing what an agent is across multiple dimensions (slot, capabilities, disposition, operational profile), grounded in empirical research rather than convention.

2. **Discovery** — a registry and vocabulary system that lets orchestrators, humans, and other agents find the right agent for a task: "find me a cautious critic with demonstrated code-review accuracy above 0.8 in Java."

3. **Generation** — a `SystemPromptRenderer` SPI that turns a descriptor + a case goal into a rendered CLAUDE.md (or any LLM system prompt). The ontology drives the generation; outcomes feed back as attestations; the knowledge graph learns which configurations work for which task types.

---

## Why a New Repo?

Eidos is **higher-order** than the ledger. It depends on casehub-ledger's attestation and trust infrastructure as its evidence layer. The ledger should never depend on it.

`casehub-ledger` is a narrow, stable foundation library — audit, tamper evidence, attestation, trust scores. Its consumers (`casehub-work`, `casehub-qhorus`, `casehub-engine`) depend on it. It stays focused.

```
casehub-platform-api       (existing — unchanged)
  └── ActorType, CurrentPrincipal, Path, PreferenceKey  (pre-existing platform primitives only)

casehub-ledger             (unchanged)
  ├── LedgerEntry, attestation, trust scores, signing, key rotation
  └── agentConfigHash already in ProvenanceSupplement (= weightsFingerprint)

eidos/                     (new repo — casehubio/eidos)
  ├── api/                 — casehub-eidos-api: pure Java, no Quarkus, no JPA (Tier 1)
  │   ├── AgentDescriptor
  │   ├── AgentCapability
  │   ├── AgentDisposition
  │   ├── Vocabulary, VocabularyTerm
  │   ├── VocabularyRegistry       (SPI)
  │   ├── AgentRegistry            (SPI — blocking)
  │   ├── ReactiveAgentRegistry    (SPI — Uni<T> variants)
  │   ├── CapabilityHealth         (SPI — blocking)
  │   ├── ReactiveCapabilityHealth (SPI — Uni<T> variants)
  │   └── SystemPromptRenderer     (SPI)
  ├── runtime/             — casehub-eidos: JPA, CDI, Quarkus (Tier 3)
  │   ├── JpaAgentRegistry          (@DefaultBean blocking)
  │   ├── JpaReactiveAgentRegistry  (@DefaultBean reactive)
  │   ├── DefaultCapabilityHealth
  │   ├── CdiVocabularyRegistry     (discovers Instance<Vocabulary> CDI beans)
  │   ├── ClaudeMarkdownRenderer    (@DefaultBean SystemPromptRenderer)
  │   └── A2AAgentCardSerializer
  ├── persistence-memory/  — casehub-eidos-memory: in-memory AgentRegistry (@Alternative @Priority(1))
  ├── deployment/          — casehub-eidos-deployment: @BuildStep Quarkus processor
  └── vocab/               — casehub-eidos-vocab: optional CDI-produced well-known vocabularies
      ├── SvoVocabulary            (@ApplicationScoped CDI bean)
      ├── ConscientiousnessVocabulary
      └── CasehubSlotVocabulary
```

**Why nothing goes in `casehub-platform-api`:** Eidos types are Eidos domain types.
Repos that need them depend on `casehub-eidos-api` (Tier 1, pure Java). The three-tier
module structure already solves the lightweight-dependency problem — no need to promote
domain types to platform-api. See the `platform-api-scope` protocol in the garden.

**Maven coordinates:**

| Module | artifactId | Folder |
|--------|-----------|--------|
| Root parent | `casehub-eidos-parent` | — |
| SPI + types | `casehub-eidos-api` | `api/` |
| Runtime | `casehub-eidos` | `runtime/` |
| In-memory (test/ephemeral) | `casehub-eidos-memory` | `persistence-memory/` |
| Quarkus deployment | `casehub-eidos-deployment` | `deployment/` |
| Well-known vocabularies (optional) | `casehub-eidos-vocab` | `vocab/` |

---

## The AgentDescriptor

A structured description of an individual LLM agent across four layers. Self-declared at registration; validated over time by attestations and trust scores.

**Key design principle:** The descriptor defines **structure** (what fields exist). Vocabularies define **values** (what goes in those fields). The platform never hardcodes vocabulary values — that is always the domain's job.

### Layer 1 — Identity

| Field | Type | Notes |
|-------|------|-------|
| `agentId` | String | Same value as `actorId` in casehub-ledger: `{model-family}:{persona}@{major}` |
| `name` | String | Human-readable display name |
| `version` | String | semver — major bump resets trust baseline |
| `provider` | String | Organisation or individual |
| `modelFamily` | String | e.g. `"claude"`, `"gpt"`, `"gemini"` |
| `modelVersion` | String | Specific model version |
| `weightsFingerprint` | String | Integrity hash — equivalent to `agentConfigHash` in `ProvenanceSupplement` (casehub-ledger), which already captures LLM config drift |

`agentId` must equal `actorId` when an agent both has a descriptor and writes ledger entries. The two identity systems must be consistent.

### Layer 2 — Slot

What position the agent fills in a team. **Open String — domain defines values, not the platform.**

```java
record AgentDescriptor(
    ...
    String slot,   // e.g. "planner", "critic", "mayor", "gatekeeper" — domain-defined
    ...
)
```

The platform provides no hardcoded constants for slot. Domain apps define their own vocabulary and register it via `VocabularyRegistry`. Optional well-known values are published in `casehub-eidos-vocab` for apps that want a starting point.

**Research backing:** MAST FM-1.2 (disobey role specification) cascades into 36.9% inter-agent misalignment. Structured slot declaration — in any vocabulary — is failure prevention, not routing convenience.

**Domain examples:**
- DevTown: `"planner"`, `"coder"`, `"reviewer"`, `"tester"`
- GasTown: `"mayor"`, `"polecat"`, `"witness"`, `"deacon"`, `"refinery"`
- Clinical: `"approver"`, `"adjudicator"`, `"supervisor"`
- Game: `"aggressor"`, `"defender"`, `"scout"`, `"support"`

### Layer 3 — Capabilities

What the agent can do. Capability names are open strings; the operational metadata is structured.

```java
record AgentCapability(
    String name,                         // open string — "code-review", "security-audit"
    double qualityHint,                  // self-declared prior 0–1; attestations update this
    Long latencyHintP50Ms,               // median latency estimate
    String costHint,                     // open string — "low", "medium", "high"
    List<String> inputTypes,             // MIME types
    List<String> outputTypes,
    List<String> tags,                   // additional classification
    Map<String, Double> epistemicDomains // sub-capability qualification per domain
                                         // e.g. {"java": 0.95, "rust": 0.42, "cobol": null}
)
```

**`qualityHint` is a self-declared prior.** The `ActorTrustScore` per `CapabilityTag` in casehub-ledger is the evidence-backed replacement that accumulates over time. LDP finding: unverified quality hints actively degrade routing quality below the no-metadata baseline. The attestation layer is not optional.

**`epistemicDomains`** — new, not present in any existing framework. A capability tag is not binary. An agent assigned a Rust PR when it has `rust: 0.42` (or null) will produce MAST FM-2.2 / FM-2.3 failures through undetected domain mismatch. This distinction is currently invisible to all registries.

**Regulatory fields** (on AgentDescriptor, not per-capability):

| Field | Type | Purpose |
|-------|------|---------|
| `jurisdiction` | String | Regulatory scope — `"EU"`, `"UK"`, `"US"` |
| `dataHandlingPolicy` | String | Data governance rules |

### Layer 4 — Disposition

How the agent approaches tasks. **All fields are open String — domain defines values via vocabulary.** All are self-declared; all are attestable from observed behaviour over time.

```java
record AgentDisposition(
    String socialOrient,    // open — e.g. "prosocial", "competitive", "ruthless", "kind"
    String ruleFollowing,   // open — e.g. "rigid", "flexible", "autonomous"
    String riskAppetite,    // open, optional — e.g. "conservative", "balanced", "aggressive"
    String autonomy,        // open — e.g. "full", "supervised", "human_in_loop"
    boolean delegation      // binary — can this agent spawn sub-agents? platform-semantic.
)
```

`delegation` is boolean because "can this agent spawn sub-agents" is binary and has platform-semantic meaning for the orchestration engine. Everything else is open String.

**Research backing for the axes:**
- `socialOrient` — Social Value Orientation (SVO): best-grounded disposition axis in the literature. Observable from allocation, cooperation, negotiation decisions. Validated in multi-agent RL and LLMs directly (2025–2026).
- `ruleFollowing` — Conscientiousness axis: predictive of cooperation and consistency. Steerable via representation engineering. Observable from rule adherence, planning, recovery from ambiguity.
- `riskAppetite` — emerging from RL exploiter roles and behavioural economics. Not yet fully formalised but practically important for routing.

**Critical finding:** behavioural consistency is weak. Self-declared disposition diverges from actual behaviour across contexts. The attestation layer is essential — a declared "prosocial" that behaves competitively accumulates FLAGGED verdicts and a degraded trust score. The descriptor is a prior; evidence updates it. This explains MAST FM-2.6 (reasoning-action mismatch, 13.2% of all failures).

D&D alignment axes were considered and dropped — they overlap with SVO and Conscientiousness with no empirical validation for AI agents.

---

## Pluggable Vocabulary System

**The platform defines structure. Domains define vocabulary.** This is the core principle.

### Vocabulary type

```java
record Vocabulary(
    String uri,           // unique identifier — "https://devtown.io/vocab/slots/v1"
    String name,          // human-readable
    String version,
    Map<String, VocabularyTerm> terms   // value → term definition
)

record VocabularyTerm(
    String value,          // e.g. "planner"
    String label,          // human-readable
    String description,
    List<String> aliases,  // e.g. ["orchestrator", "coordinator"] — for cross-vocab discovery
    Map<String, String> exactMatches  // URI → equivalent term in another vocabulary
)
```

### VocabularyRegistry SPI

```java
// in casehub-eidos-api
interface VocabularyRegistry {
    void register(Vocabulary vocabulary);
    Optional<Vocabulary> find(String uri);
    Optional<VocabularyTerm> resolve(String vocabularyUri, String value);
    Set<String> equivalentValues(String vocabularyUri, String value, String targetVocabularyUri);
}
```

Apps register vocabularies at startup. The registry resolves equivalences between vocabularies for cross-vocabulary discovery.

### Domain profile on AgentDescriptor

```java
record AgentDescriptor(
    // identity...
    String domainVocabulary,       // default vocabulary URI — covers all fields
    String slotVocabulary,         // optional override for slot only
    String dispositionVocabulary,  // optional override for all disposition fields
    String slot,
    List<AgentCapability> capabilities,
    AgentDisposition disposition,
    // regulatory...
)
```

**Common case — one line:**
```yaml
domainVocabulary: "https://devtown.io/vocab/v1"
slot: "planner"
disposition:
  socialOrient: "methodical"
  ruleFollowing: "flexible"
  autonomy: "supervised"
  delegation: false
```

**Override case — mix vocabularies:**
```yaml
domainVocabulary: "https://devtown.io/vocab/v1"
dispositionVocabulary: "https://eidos.io/vocab/svo/v1"   # use SVO for disposition
slot: "planner"
disposition:
  socialOrient: "prosocial"    # SVO vocabulary value
  ruleFollowing: "flexible"    # SVO vocabulary value
```

**No vocabulary — raw strings, exact match only:**
```yaml
slot: "my-custom-slot"   # no vocabulary; exact match only in discovery
```

### Cross-vocabulary discovery

If DevTown's `"planner"` declares `exactMatch: "https://casehub.io/vocab/slots/v1#orchestrator"`, then querying for slot = "orchestrator" in the registry can surface DevTown agents with slot = "planner". The registry resolves the equivalence; the querying agent doesn't need to know DevTown's vocabulary.

### The optional `casehub-eidos-vocab` module

Provides well-known starting-point vocabularies. **Not required.** Apps that want their own vocabulary don't use this. Apps that want a shared baseline can depend on it.

- `CasehubSlotVocabulary` — orchestrator, executor, critic, monitor, synthesiser, specialist
- `SvoVocabulary` — altruist, prosocial, individualist, competitive
- `ConscientiousnessVocabulary` — rigid, flexible, autonomous
- `RiskAppetiteVocabulary` — conservative, balanced, aggressive

---

## Two-Layer Capability Architecture

The `AgentDescriptor` is the **static layer** — what an agent IS and CAN DO. Declared at registration, versioned, stored. But a declared capability is not the same as an operable one.

No existing framework has formalised this for LLM agents. Robotics acknowledges the gap (Dussard 2023). Microservices have Kubernetes probes but binary. MAST has no failure mode for "agent declared capable but not currently operable."

Eidos introduces `CapabilityHealth` — the **dynamic layer**, probed at dispatch time, TTL-limited, graduated:

```
READY | DEGRADED(reason) | UNAVAILABLE | EPISTEMICALLY_WEAK
```

Degradation reasons: `RATE_LIMITED`, `CONTEXT_EXHAUSTED`, `OVERLOADED`, `DOMAIN_MISMATCH`.

### Dispatch flow (casehub-engine integration)

```
WorkerProvisioner.getCapabilities()   → static filter  (declared capable?)
TrustGateService.meetsThreshold()     → trust gate     (historically trustworthy?)
CapabilityHealth.probe()              → health check   (operable right now?)
      │
      ├── READY               → schedule
      ├── DEGRADED            → schedule with warning; log degradation reason to ledger
      ├── UNAVAILABLE         → skip; try next candidate
      └── EPISTEMICALLY_WEAK  → skip unless no better candidate; log domain mismatch
      
If all candidates fail → WorkerProvisioner.provision() with epistemic constraints
```

### `CapabilityHealth` SPI

```java
// in casehub-eidos-api
interface CapabilityHealth {
    CapabilityStatus probe(String agentId, String capabilityTag, ProbeContext context);
}

record ProbeContext(String taskDomain, Map<String, Object> taskMetadata) {}

sealed interface CapabilityStatus permits
    CapabilityStatus.Ready,
    CapabilityStatus.Degraded,
    CapabilityStatus.Unavailable,
    CapabilityStatus.EpistemicallyWeak {

    record Ready() implements CapabilityStatus {}
    record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
    record Unavailable(String reason) implements CapabilityStatus {}
    record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
}
```

Default implementation checks context utilisation, rate limit headroom, and `epistemicDomains` from the descriptor. Consumers override with platform-specific signals.

### New MAST failure class

MAST taxonomy has no failure mode for capability operability failure. Eidos logging of `CapabilityHealth.probe()` results creates the observability to distinguish:
- **Genuine misalignment** (FM-2.2 / FM-2.3): agent received appropriate task, miscoordinated
- **Operability failure** (new): agent was dispatched despite degraded state; preventable at routing time

---

## The Generative Loop

The `AgentDescriptor` is not only a discovery artifact — it is a **generator**.

```
CaseDefinition.Goal  ──┐
AgentDescriptor       ──┼──▶  SystemPromptRenderer  ──▶  rendered CLAUDE.md
  (all four layers)    │                                          │
  + context string     │                                          ▼
                       │                                    run agent
                       │                                          │
                       │                                          ▼
                       │                                    outcomes + attestations
                       │                                          │
                       └──────────── update trust scores / knowledge graph
```

**The goal comes from the Case** — `CaseDefinition.Goal` is a formal evaluable expression in casehub-engine. You read it; you don't invent it.

**Same descriptor, different context → different agent:**

*Goal:* "Review this PR. Accept only if code quality and test coverage meet the team standard."

| Descriptor | Context | Rendered disposition |
|-----------|---------|---------------------|
| `slot: "critic", socialOrient: "prosocial", ruleFollowing: "flexible"` | `"reviewer of intern submissions"` | "Be skeptical but kind — your role is coaching, not gatekeeping." |
| `slot: "critic", socialOrient: "competitive", ruleFollowing: "rigid"` | `"gatekeeper for production branch"` | "Standards are non-negotiable. Reject anything that wastes the team's time." |

Same goal. Same capability. Radically different behaviour from two structured dimension choices.

**Systematic experimentation:** hold the goal constant, vary the disposition, observe outcomes. The knowledge graph accumulates `(descriptor config, task type, outcome, attestation)` tuples — institutional memory for which configurations work for which task types.

---

## SystemPromptRenderer SPI

Format-agnostic from day one. Current target: CLAUDE.md.

```java
// in casehub-eidos-api
interface SystemPromptRenderer {
    RenderedPrompt render(AgentDescriptor descriptor, Goal goal, RenderContext context);
}

record RenderContext(
    String situationalContext,   // e.g. "reviewer of intern submissions"
    RenderFormat format,         // CLAUDE_MD | OPENAI_SYSTEM | A2A_CARD | GEMINI
    Locale locale
)
```

| Implementation | Format | Notes |
|---|---|---|
| `ClaudeMarkdownRenderer` | CLAUDE.md | `@DefaultBean` — current target |
| `OpenAISystemPromptRenderer` | OpenAI system message | Future |
| `A2AAgentCardRenderer` | A2A Agent Card JSON | Future |
| `GeminiSystemPromptRenderer` | Gemini system instruction | Future |

**Hybrid rendering:**
- **Template layer (Qute):** structural skeleton — slot header, goal injection, capability list, format-specific structure. Deterministic.
- **LLM prose layer:** disposition section only — renders socialOrient + ruleFollowing + situationalContext into 2–3 natural language sentences. Lightweight LLM call at descriptor registration time, not at runtime. Cached by `(descriptorHash + contextHash)`.

The CLAUDE.md structure lives entirely inside `ClaudeMarkdownRenderer`. The descriptor and SPI know nothing about it.

**Rendering unit:** a structured *section* for inclusion in the agent's CLAUDE.md alongside repo-specific content — not a full file replacement.

---

## Ecosystem Position

### What already exists

| Layer | Who | What it has | Gap |
|-------|-----|-------------|-----|
| Capability advertisement | A2A Agent Cards | Skills, input/output types, auth schemes | No slot, no disposition, no trust evidence |
| Model identity | LDP (arXiv:2603.08852) | Model family, quality hints, one-dimensional freeform reasoning profile | Hints designer-assigned; profile not structured |
| Agent naming | OWASP ANS | Protocol + capability + provider + version | No behavioural dimensions |
| Agent auth | OIDC-A | Delegation chains, attestation claims | Identity only |
| Failure evidence | MAST (NeurIPS 2025) | 36.9% failures are inter-agent misalignment | No structural fix; fixes are prompt-based |
| Reputation | A2A Discussion #1631 | Multi-party proposals, no standard yet | Fragmented; four competing proprietary approaches |

### Ecosystem fragmentation

Discussion #1631 shows four independent proprietary implementations racing: ARP (custom trust substrate), MoltBridge (Neo4j graph), laplace0x (ERC-8004 Ethereum), and dispute resolution proposals. None coordinating on a shared standard. Classic early-market fragmentation.

The standard is not settled. The window to influence it is open.

casehub's posture differs from all of them: open standards (A2A extension mechanism, JWS, PROV-O), contribute back rather than proprietary lock-in.

### A2A compatibility — fully extensible, no breaking changes

A2A spec v1.0 explicitly states clients MUST ignore unrecognised fields.

**Mechanism 1 — `extensions[]`** (schema declaration, vocabulary URIs not hardcoded values):
```json
{
  "capabilities": {
    "extensions": [{
      "uri": "https://eidos.io/extensions/v1/agent-dimensions",
      "required": false,
      "params": {
        "slotVocabulary": "https://devtown.io/vocab/slots/v1",
        "dispositionVocabulary": "https://eidos.io/vocab/svo/v1"
      }
    }]
  }
}
```

**Mechanism 2 — `metadata` maps** (actual values, namespaced):
```json
{
  "metadata": {
    "eidos_slot": "planner",
    "eidos_slot_vocabulary": "https://devtown.io/vocab/slots/v1",
    "eidos_disposition_social_orient": "prosocial",
    "eidos_epistemic_domains": {"java": 0.95, "rust": 0.42}
  }
}
```

**Mechanism 3 — Extended authenticated card** — `extendedAgentCard: true` for richer profiles served to authenticated clients. Full `CapabilityHealth` state and `epistemicDomains` detail live here.

**JWS signatures cover metadata** — slot and disposition values are cryptographically bound by default.

**Contribution target:** A2A Discussion #1631 — engage when Eidos is built. Bring SVO vocabulary, slot taxonomy, and the connection between self-declared dimensions and attestation-backed trust scores.

### casehub-ledger as the evidence layer

Eidos self-declared dimensions are priors. The evidence layer that validates or challenges them is already built in casehub-ledger:

| Eidos self-declaration | casehub-ledger evidence |
|------------------------|------------------------|
| `qualityHint: 0.85` per capability | `ActorTrustScore` per `CapabilityTag` (Bayesian Beta, attestation-derived) |
| `weightsFingerprint` | `agentConfigHash` in `ProvenanceSupplement` (already exists) |
| `socialOrient: "prosocial"` | Peer attestation verdicts (SOUND/FLAGGED/ENDORSED/CHALLENGED) |
| `delegation: true` | Causal chain via `causedByEntryId` in ledger entries |

---

## What Eidos Is NOT

- **Not a communication protocol** — that's A2A / MCP / Qhorus
- **Not an orchestration engine** — that's casehub-engine
- **Not an audit log** — that's casehub-ledger (Eidos depends on it)
- **Not a trust scoring system** — that's casehub-ledger's `TrustScoreComputer` / `EigenTrustComputer`
- **Not a replacement for CLAUDE.md** — `ClaudeMarkdownRenderer` generates a *section* for inclusion alongside repo-specific content
- **Not a vocabulary authority** — platform defines structure; domains define vocabulary values

---

## Key Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| New repo vs. evolve ledger | New repo | Ledger is narrow, stable, foundation; Eidos is higher-order and depends on it |
| Type location | `casehub-eidos-api` (Tier 1) | Platform-api is for platform primitives only; repos needing Eidos types depend on `casehub-eidos-api` directly |
| Slot type | Open `String`, no platform constants | Domain apps define their own vocabulary; platform must not constrain |
| Disposition field types | Open `String`, no platform constants | Same — "ruthless", "kind", "methodical" are domain-valid values |
| `delegation` type | `boolean` | Binary, platform-semantic — orchestration engine needs this regardless of domain |
| Vocabulary model | Pluggable `VocabularyRegistry` SPI; domain profiles with per-field overrides | Fluid, evolvable; `domainVocabulary` default + optional `slotVocabulary` / `dispositionVocabulary` overrides |
| Optional vocab module | `casehub-eidos-vocab` | Recommended starting vocabulary; not required; SVO, Conscientiousness, CasehubSlot |
| Disposition model basis | SVO + Conscientiousness + Risk appetite | Empirically grounded; replaces D&D which has no academic validation for AI agents |
| Rendering strategy | Hybrid: Qute template + LLM prose | Template for deterministic structure; LLM for natural language disposition section only |
| Rendering timing | Registration time, cached | Determinism, cost, auditability — stored artifact, regenerated on descriptor change |
| Quality hints | Self-declared prior + attestation update | LDP: unverified hints degrade quality below baseline; attestation layer is not optional |
| Heavy semantic stack | Excluded from runtime | Jena, RDF4J, JADE — out of scope; ontologies inform vocabulary, not runtime deps |
| A2A engagement timing | Build first, engage later | Go full steam ahead; engage from position of strength with a working implementation |

---

## Roadmap — Tentative

Target: one week to get the foundations in place. Four phases, iterating.

### Phase 1 — The Descriptor (tonight → day 2–3)

Get the core types into `casehub-eidos-api` and a working registry in `casehub-eidos` runtime.

- `AgentDescriptor` record — all four layers (identity, slot, capabilities, disposition)
- `AgentCapability` record — name, qualityHint, latencyHint, costHint, inputTypes, outputTypes, tags, epistemicDomains
- `AgentDisposition` record — open String fields + delegation boolean
- `Vocabulary` and `VocabularyTerm` types
- `VocabularyRegistry` SPI + in-memory default implementation
- `AgentRegistry` SPI — register, findById, findBySlot, findByCapability
- JPA `AgentDescriptorEntity` in Eidos runtime
- `JpaAgentRegistry` implementation
- `casehub-eidos-vocab` module with SVO, Conscientiousness, CasehubSlot vocabularies
- A2A Agent Card serialisation (extensions + metadata, vocabulary URIs)
- Basic Flyway migration for `agent_descriptor` table

*Deliverable: devtown and claudony can register agents and fetch descriptors.*

### Phase 2 — Discovery + Health (day 3–5)

Make the registry useful for routing decisions.

- `CapabilityHealth` SPI + sealed `CapabilityStatus` type
- `DefaultCapabilityHealth` implementation — checks epistemicDomains, rate limit headroom, context availability
- Discovery queries: findBySlot, findByCapability, findByDisposition, findByTrustThreshold
- Cross-vocabulary discovery via `VocabularyRegistry.equivalentValues()`
- casehub-engine integration — `CapabilityHealth.probe()` wired into dispatch flow after `TrustGateService`
- CapabilityHealth probe results logged to casehub-ledger for operability failure observability

*Deliverable: casehub-engine can route to the right agent, accounting for trust, health, and epistemic domain.*

### Phase 3 — Generation (day 5–7)

Turn descriptors into runnable agent instructions.

- `SystemPromptRenderer` SPI + `RenderContext` record in `casehub-eidos-api`
- `ClaudeMarkdownRenderer` — `@DefaultBean`, Qute template for structure
- LLM prose call for disposition section — lightweight call at registration time
- Goal injection from `CaseDefinition.Goal`
- Rendered prompt cache keyed by `(descriptorHash + contextHash)`
- `RenderedPrompt` storage in Eidos (text + metadata + rendering timestamp)
- `RenderFormat` enum: `CLAUDE_MD`, `OPENAI_SYSTEM`, `A2A_CARD`, `GEMINI`

*Deliverable: given a case goal and a descriptor, get a CLAUDE.md section back.*

### Phase 4 — Learning Layer (following week)

Close the feedback loop.

- `DescriptorOutcome` entity — links descriptor config + task type + attestation result
- `AgentKnowledgeGraph` service — query: which configurations work for which task types
- Descriptor recommendation: given a task type, suggest descriptor configurations with track record
- Dashboard / query API for human-readable insight

*Deliverable: the system learns which agent configurations work for which task types.*

---

## Open Questions

**Architecture:**
- Name is Eidos — decided.
- Does `ClaudeMarkdownRenderer` live in `eidos/runtime` or a separate `casehub-eidos-claude` opt-in module?
- Which LLM generates the disposition prose? Platform-configured or descriptor-specified?

**Descriptor design:**
- Should `jurisdiction` + `dataHandlingPolicy` be on `AgentDescriptor` directly, or a separate compliance supplement parallel to `ComplianceSupplement` in the ledger?
- How many capability entries before the descriptor becomes unwieldy? Is there a max?
- Should `epistemicDomains` values be expressed as confidence floats (0–1) or as a richer type?

**Vocabulary system:**
- How does vocabulary versioning work? If `svo/v1` terms change, do all descriptors using it need re-registration?
- Should vocabulary terms support `broaderMatch` / `narrowerMatch` (SKOS-like) in addition to `exactMatch`?
- Who can publish a vocabulary? Is the registry open or governed?

**Discovery:**
- Query model: simple field match (Phase 1–2), then embedding similarity for semantic queries (Phase 4+)?
- How does Eidos registry relate to A2A Agent Card registries at `/.well-known/`? Separate concerns or federated?

**Evidence loop:**
- What outcome signals feed the knowledge graph? Attestation verdicts + quantitative metrics (PR merge rate, cycle time)?
- Staleness policy for CapabilityHealth cache: `fail-closed` (reject if stale) vs `fail-open` (use last known) vs `deferred-audit`?

**Anti-gaming:**
- Sybil attacks (synthetic agent networks rating each other) — open for everyone in this space
- Trust-building-then-exploit — addressed by casehub-ledger's key rotation + SUSPECT detection at the identity layer; collusion between real agents is still open

---

## What's Left to Research

- [x] LDP (arXiv:2603.08852) — schema documented; quality hints designer-assigned; weightsFingerprint = agentConfigHash; unverified hints harmful
- [x] MAST (arXiv:2503.13657) — 36.9% inter-agent misalignment; FM-2.6 (13.2%) = reasoning-action mismatch; no operability failure mode
- [x] Disposition thread — SVO replaces D&D; consistency weak; attestation essential; Nature MI 2025
- [x] Feature vs. capability — two-layer: static descriptor + dynamic CapabilityHealth; epistemicDomains; new MAST failure class
- [x] A2A extensibility — fully compatible; extensions[] + metadata + extended card; JWS covers metadata; Discussion #1631 is contribution target
- [ ] A2A Discussion #741 — registry/federation; where Eidos registry plugs in to the A2A ecosystem
- [ ] OIDC-A delegation chains — trust propagation when agent A delegates to agent B
- [ ] WoT Thing Description Directory — most mature discovery; SPARQL query model worth understanding for Phase 2+
- [ ] Risk appetite formalisation — any emerging vocabulary from behavioural economics / RL?
