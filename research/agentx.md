# AgentX — Specification

**Status:** Research / Pre-spec  
**Started:** 2026-05-22  
**Working name:** AgentX (permanent name TBD)  
**Research backing:** `agent-description-ontology.md`, `casehub-platform-vocabulary-validation.md`

---

## What Is AgentX?

AgentX is a new casehub platform repo that gives LLM agents a structured, discoverable, and generative identity.

Three things it enables:

1. **Description** — a structured `AgentDescriptor` that captures what an agent is across multiple dimensions (role, capabilities, disposition, operational profile), grounded in empirical research rather than convention.

2. **Discovery** — a registry that lets orchestrators, humans, and other agents find the right agent for a task: "find me a cautious critic with demonstrated code-review accuracy above 0.8."

3. **Generation** — a `SystemPromptRenderer` SPI that turns a descriptor + a case goal into a rendered CLAUDE.md (or any LLM system prompt). The ontology drives the generation; outcomes feed back as attestations; the knowledge graph learns which configurations work for which task types.

---

## Why a New Repo?

AgentX is **higher-order** than the ledger. It depends on casehub-ledger's attestation and trust infrastructure as its evidence layer. The ledger should never depend on it.

`casehub-ledger` is a narrow, stable foundation library — audit, tamper evidence, attestation, trust scores. Its consumers (`casehub-work`, `casehub-qhorus`, `casehub-engine`) depend on it. It stays focused.

AgentX sits above that:

```
casehub-platform-api
  ├── AgentDescriptor          (type — consumed by ledger, AgentX, and consumers)
  └── SystemPromptRenderer     (SPI — implemented in AgentX)

casehub-ledger                 (unchanged)
  ├── LedgerEntry, attestation, trust scores, signing, key rotation
  └── depends on casehub-platform-api (already does)

agentx                         (new repo)
  ├── AgentRegistry            (store + discovery)
  ├── ClaudeMarkdownRenderer   (@DefaultBean SystemPromptRenderer)
  ├── LLM prose renderer       (disposition → natural language section)
  └── Knowledge graph          (descriptor × task × outcome × attestation)
       depends on → casehub-ledger  (evidence layer)
       depends on → casehub-platform-api  (types)
```

`AgentDescriptor` and `SystemPromptRenderer` go in `casehub-platform-api` because they are fundamental types other repos may reference without pulling in the full AgentX implementation.

---

## The AgentDescriptor

A structured description of an individual LLM agent across four layers. Self-declared at registration; validated over time by attestations and trust scores.

### Layer 1 — Identity

| Field | Type | Notes |
|-------|------|-------|
| `agentId` | URI / structured name | Extends existing `actorId` format: `{model-family}:{persona}@{major}` |
| `name` | string | Human-readable |
| `version` | semver | Major bump = trust baseline reset |
| `provider` | string | Organisation or individual |
| `modelFamily` | string | e.g. `"claude"`, `"gpt"`, `"gemini"` |
| `modelVersion` | string | Specific model version |
| `weightsFingerprint` | string | Integrity hash — equivalent to `agentConfigHash` already in `ProvenanceSupplement` |

`actorId` already handles identity in casehub-ledger. `AgentDescriptor.agentId` extends it; they must be the same value when an agent both has a descriptor and writes ledger entries.

### Layer 2 — Functional Role

A closed vocabulary defining what *slot* the agent fills in a team.

| Value | Meaning |
|-------|---------|
| `orchestrator` | Decomposes goals, delegates to others, integrates results |
| `executor` | Carries out a specific well-defined task |
| `critic` | Reviews, evaluates, gatekeeps |
| `monitor` | Observes, detects anomalies, reports |
| `synthesiser` | Combines outputs from multiple sources |
| `specialist` | Deep capability in a narrow domain |

**Research backing:** MAST FM-1.2 (disobey role specification) cascades into 36.9% inter-agent misalignment. Undefined roles are a structural failure mode, not a model limitation.

**Game AI analogue:** D&D party roles (Defender/Striker/Leader/Controller). GasTown analogue: Mayor/Polecat/Witness/Deacon/Refinery.

### Layer 3 — Capabilities

What the agent can do — open vocabulary of skills with operational metadata per skill.

```yaml
capabilities:
  - name: code-review
    qualityHint: 0.85        # self-declared prior; attestations update this
    latencyHintP50Ms: 8000
    costHint: medium
    inputTypes: [text/diff, text/markdown]
    outputTypes: [text/markdown]
    tags: [java, security, quarkus]
    epistemicDomains:        # sub-capability qualification — declared confidence per domain
      java: 0.95
      kotlin: 0.88
      rust: 0.42
      cobol: null            # unknown — agent has not operated in this domain
  - name: security-audit
    qualityHint: 0.70
    ...
```

`qualityHint` is a self-declared prior. The `ActorTrustScore` per `CapabilityTag` in casehub-ledger is the evidence-backed replacement that accumulates over time. **LDP finding:** unverified quality hints actively degrade routing quality below the no-metadata baseline. The attestation layer is not optional.

**`epistemicDomains`** is a new dimension not present in any existing framework. A capability tag is not binary — "code-review" has domain qualifications. An agent assigned a Rust PR when it has `rust: 0.42` (or null) will produce MAST FM-2.2 / FM-2.3 failures through undetected domain mismatch, not genuine misalignment. This distinction is currently invisible to all registries.

**Operational fields** (`latencyHintP50Ms`, `costHint`) come from LDP — practically important for routing decisions.

**Regulatory fields:**
| Field | Purpose |
|-------|---------|
| `jurisdiction` | Regulatory scope (e.g. `"EU"`, `"UK"`, `"US"`) |
| `dataHandlingPolicy` | Data governance rules |

### Layer 4 — Behavioural Disposition

How the agent approaches tasks. Three axes, all self-declared, all attestable from observed behaviour.

```yaml
disposition:
  socialOrient: prosocial          # altruist | prosocial | individualist | competitive
  ruleFollowing: flexible          # rigid | flexible | autonomous
  riskAppetite: conservative       # conservative | balanced | aggressive (optional)
  autonomy: supervised             # full | supervised | human_in_loop
  delegation: true                 # can this agent spawn sub-agents?
```

**`socialOrient` — Social Value Orientation (SVO)**  
Best-grounded disposition axis in the literature. Validated in behavioural economics, multi-agent RL, and LLMs directly (2025–2026). Observable from allocation, cooperation, and negotiation decisions. Replaces the earlier D&D-inspired `otherWeight` axis.

**`ruleFollowing` — Conscientiousness axis**  
Predictive of cooperation and consistency in LLMs. Steerable via representation engineering. Observable from rule adherence, planning behaviour, recovery from ambiguity.

**`riskAppetite`** — Emerging from RL exploiter roles and behavioural economics. Not yet fully formalised but practically important for routing (conservative agent for high-stakes; aggressive for open-ended exploration).

**Critical research finding:** behavioural consistency is weak. Self-declared disposition diverges from actual behaviour across contexts. This makes the attestation layer essential — a declared `prosocial` that behaves competitively in practice will accumulate FLAGGED verdicts and a degraded trust score. The descriptor is a prior; evidence updates it. This also explains MAST FM-2.6 (reasoning-action mismatch, 13.2% of all failures).

**D&D alignment dropped:** good–evil ≈ SVO; law–chaos ≈ Conscientiousness. No empirical validation for AI agents. Overlaps with better-grounded frameworks.

---

## Two-Layer Capability Architecture

The `AgentDescriptor` is the **static layer** — what an agent IS and CAN DO. Declared at registration, versioned, stored in the registry. But a declared capability is not the same as an operable one.

**No existing framework has formalised this for LLM agents.** Robotics acknowledges the gap (Dussard 2023 explicitly flags it as future work). Microservices have Kubernetes probes but they're binary. MAST has no failure mode for "agent declared capable but not currently operable" — FM-2.2 and FM-2.3 (11.65% + 7.15% of failures) can both be produced by operability failures that MAST currently attributes to misalignment.

AgentX introduces a `CapabilityHealth` **dynamic layer** — probed at dispatch time, TTL-limited, graduated:

```
READY | DEGRADED(reason) | UNAVAILABLE | EPISTEMICALLY_WEAK
```

Degradation reasons: `RATE_LIMITED`, `CONTEXT_EXHAUSTED`, `OVERLOADED`, `DOMAIN_MISMATCH`.

### Dispatch flow in casehub-engine

```
WorkerProvisioner.getCapabilities()   → static filter  (declared capable?)
TrustGateService.meetsThreshold()     → trust gate     (historically trustworthy?)
CapabilityHealth.probe()              → health check   (operable right now?)
      │
      ├── READY           → schedule
      ├── DEGRADED        → schedule with warning / log degradation reason
      ├── UNAVAILABLE     → skip; try next candidate
      └── EPISTEMICALLY_WEAK → skip unless no better candidate; log domain mismatch
```

If all candidates fail health check → `WorkerProvisioner.provision()` to spin up a new instance.

### The `CapabilityHealth` SPI

```java
// in casehub-platform-api
interface CapabilityHealth {
    CapabilityStatus probe(String agentId, String capabilityTag, ProbeContext context);
}

record ProbeContext(String taskDomain, Map<String, Object> taskMetadata) {}

sealed interface CapabilityStatus {
    record Ready() implements CapabilityStatus {}
    record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
    record Unavailable(String reason) implements CapabilityStatus {}
    record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
}
```

Default implementation: checks context utilisation, rate limit headroom, and `epistemicDomains` from the descriptor against the requested task domain. Consumers override for platform-specific health signals (e.g., hitting the Claude API to check session state).

### New MAST failure class

The MAST taxonomy (NeurIPS 2025) has no failure mode for capability operability failure. AgentX logging of `CapabilityHealth.probe()` results — including degradation reasons and domain mismatch decisions — creates the observability to distinguish:

- **Genuine misalignment** (FM-2.2 / FM-2.3): agent received appropriate task, miscoordinated
- **Operability failure** (new): agent was dispatched despite degraded state; failure was preventable at routing time

This is the first framework to make this distinction observable.

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

**The goal comes from the Case** — `CaseDefinition.Goal` is already a formal evaluable expression in casehub-engine. You read it; you don't invent it.

**Same descriptor, different context → different agent:**

*Goal:* "Review this PR. Accept only if code quality and test coverage meet the team standard."

| Descriptor | Context | Rendered disposition |
|-----------|---------|---------------------|
| `slot: critic, socialOrient: prosocial, ruleFollowing: flexible` | `"reviewer of intern submissions"` | "Be skeptical but kind — your role is coaching, not gatekeeping. Adapt your feedback to the author's level." |
| `slot: critic, socialOrient: competitive, ruleFollowing: rigid` | `"gatekeeper for production branch"` | "Standards are non-negotiable. Reject anything that wastes the team's time. Do not soften feedback." |

Same goal. Same capability. Radically different behaviour from two structured dimension choices.

**What this enables:** systematic experimentation. Hold the goal constant. Vary the disposition. Observe outcomes. The knowledge graph accumulates `(descriptor config, task type, outcome, attestation)` tuples — institutional memory for which configurations work for which task types.

---

## SystemPromptRenderer SPI

Designed for the current target (CLAUDE.md) but extensible to any LLM format from day one.

```java
// in casehub-platform-api
interface SystemPromptRenderer {
    RenderedPrompt render(AgentDescriptor descriptor, Goal goal, RenderContext context);
}

record RenderContext(String situationalContext, RenderFormat format, Locale locale) {}
```

Implementations:

| Implementation | Format | Notes |
|---------------|--------|-------|
| `ClaudeMarkdownRenderer` | CLAUDE.md | `@DefaultBean` — current target |
| `OpenAISystemPromptRenderer` | OpenAI system message | Future |
| `A2AAgentCardRenderer` | A2A Agent Card JSON | Future |
| `GeminiSystemPromptRenderer` | Gemini system instruction | Future |

**Hybrid rendering within each renderer:**

- **Template layer (Qute):** structural skeleton — role header, goal injection, capability list, format-specific sections. Fully deterministic.
- **LLM prose layer:** disposition section only — renders SVO + conscientiousness + context into 2–3 sentences of natural language behavioural instruction. Lightweight LLM call at *descriptor registration time*, not at runtime. Cached by `(descriptorHash + contextHash)`.

The CLAUDE.md-specific structure lives entirely inside `ClaudeMarkdownRenderer`. The `AgentDescriptor` and SPI contract know nothing about CLAUDE.md format.

---

## Ecosystem Position

### What already exists

| Layer | Who | What it has | Gap |
|-------|-----|-------------|-----|
| Capability advertisement | A2A Agent Cards | Skills, input/output types, auth schemes | No role slot, no disposition, no trust evidence |
| Model identity | LDP (arXiv:2603.08852) | Model family, quality hints, reasoning profile | Profile is one-dimensional freeform; hints are designer-assigned |
| Agent naming | OWASP ANS, A2A registry | Protocol + capability + provider + version | No behavioural dimensions |
| Agent auth | OIDC-A | Delegation chains, attestation claims | Identity only, not capability or disposition |
| Failure evidence | MAST (NeurIPS 2025) | 36.9% of failures are inter-agent misalignment | No structural fix — their fixes are prompt-based |

### Where AgentX sits

AgentX is the first system to combine:
- **Structured multi-dimensional description** (role + capability + disposition) with closed vocabularies grounded in empirical research
- **Evidence-backed trust** connecting self-declared dimensions to peer attestations over time
- **Generative capability** turning structured descriptors into rendered system prompts
- **Feedback loop** from outcomes back to descriptor refinement via the knowledge graph

No existing framework has all four. A2A has description (partial). casehub-ledger has evidence. Nobody has the generative loop or the feedback.

### A2A compatibility

`AgentDescriptor` can serialise as an A2A Agent Card extension — interoperable with the ecosystem while adding dimensions A2A intentionally omits. `ClaudeMarkdownRenderer` is unrelated to A2A; the `A2AAgentCardRenderer` implementation handles the wire format.

### Contribution opportunities

| Target | What to contribute |
|--------|-------------------|
| AgentO ontology | Functional role slot, SVO disposition axes, Goal formalism, Milestone, normative layer |
| A2A ecosystem | Extended Agent Card schema with role + disposition dimensions |
| OASIS W3C CG | LLM-specific agent extensions |

---

## What AgentX Is NOT

- **Not a communication protocol** — that's A2A / MCP / Qhorus
- **Not an orchestration engine** — that's casehub-engine
- **Not an audit log** — that's casehub-ledger (though AgentX depends on it)
- **Not a trust scoring system** — that's casehub-ledger's `TrustScoreComputer` / `EigenTrustComputer`
- **Not a replacement for CLAUDE.md** — `ClaudeMarkdownRenderer` generates a structured *section* that agents include in their CLAUDE.md alongside repo-specific content

---

## Key Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| New repo vs. evolve ledger | New repo | Ledger is a narrow stable foundation dependency; AgentX is higher-order |
| AgentDescriptor location | `casehub-platform-api` | Fundamental type; other repos may reference without pulling AgentX |
| Disposition model | SVO + Conscientiousness + Risk appetite | Empirically grounded; replaces D&D alignment which has no academic validation |
| Rendering strategy | Hybrid: Qute template + LLM prose | Template for deterministic structure; LLM for natural language disposition section |
| Rendering timing | Registration time, not runtime | Determinism, cost, auditability — stored artifact, regenerated on descriptor change |
| Quality hints | Self-declared prior + attestation update | LDP: unverified hints degrade quality below baseline; attestation layer is essential |
| Heavy semantic stack | Excluded | Jena, RDF4J, JADE — out of scope; ontologies inform vocabulary only, not runtime deps |

---

## Open Questions

**Architecture:**
- What is the right permanent name? AgentX is a working name.
- Which module within AgentX owns `ClaudeMarkdownRenderer`? Separate `agentx-claude` module for opt-in?
- How does the knowledge graph store `(descriptor, task, outcome)` tuples? JPA? Separate module?

**Descriptor design:**
- How does `AgentDescriptor.agentId` relate to `actorId` in casehub-ledger? Same value? Derived from it?
- Should `jurisdiction` + `dataHandlingPolicy` be part of the descriptor or a separate compliance supplement?
- How many capability entries before the descriptor becomes unwieldy?

**Rendering:**
- Which LLM generates the disposition prose? Platform-configured or descriptor-specified?
- What is the rendering unit — full CLAUDE.md or a structured section to be included?
- How does `situationalContext` (e.g. `"coaching interns"`) get supplied — from the Case, from the deployer, from the agent?

**Evidence loop:**
- What outcome signals feed the knowledge graph? Attestation verdicts only? Quantitative metrics (PR merge rate, cycle time)?
- How many `(descriptor config, task type)` observations before the graph can recommend configurations?
- Does the knowledge graph live in AgentX or is it a separate analytical layer?

**Discovery:**
- What query model? Tag matching (simple), SPARQL (powerful, heavy), natural language + embedding similarity (flexible)?
- How does AgentX registry relate to A2A Agent Card registries? Federated? Separate?

---

## What's Left to Research

- [x] Feature vs. capability distinction — done. Two-layer architecture: static `AgentDescriptor` + dynamic `CapabilityHealth` SPI. `epistemicDomains` added to capability layer. New MAST failure class identified.
- [ ] A2A Agent Card extensibility — can we add role + disposition without breaking compatibility?
- [ ] A2A Discussion #741 — registry/federation; where AgentX registry plugs in
- [ ] OIDC-A delegation chains — how trust propagates when agent A delegates to agent B
- [ ] WoT Thing Description Directory — most mature discovery implementation; SPARQL query model
- [ ] MAST-Data dataset (1,642 traces) — potential validation for whether typed role/disposition reduces FC2 failure rates
- [ ] Risk appetite formalisation — any standard vocabulary emerging?
