# Agent Description Ontology — Research Notes

**Status:** Supporting research — see [`eidos.md`](eidos.md) for the synthesised spec  
**Started:** 2026-05-22  
**Destination:** `casehub-parent/docs/research/` when mature  

---

## The Problem

We can identify an agent (`actorId`), sign its entries (Ed25519), track its trust over time (Bayesian Beta per capability), and detect key compromise. What we cannot do is *describe* what kind of agent something is in a way that makes it discoverable.

Today, all of that meaning is implicit in the `actorId` string: `"claude:reviewer@v1"`. There is no machine-readable way to ask: "find me an agent that can do X", "find me a cautious agent for a high-stakes task", or "which agents in this mesh are orchestrators vs. workers?"

This is the gap. The research below maps the landscape and defines what a solution might look like.

---

## Motivating Examples

These are the cases that should be solvable:

- **DevTown**: spawn a team of LLM agents each with a different role (planner, coder, reviewer, tester). An orchestrator needs to discover which agent fills which slot without hardcoding endpoints.
- **GasTown**: Mayor, Polecat, Witness, Deacon, Refinery — each a distinct functional role in a multi-agent coding workflow. Roles need to be introspectable by other agents.
- **StarCraft 2 team**: a set of LLMs each assigned a unit role (aggressor, defender, scout, support). Team composition is an input; agents need to know what their teammates can do.
- **Game alignment**: a player-facing agent can be configured as ruthless or kind. That disposition affects how it behaves and what tasks it should be given. The platform needs to express and honour that setting.
- **Trust-gated delegation**: before delegating to an agent, an orchestrator checks not just "can this agent do code review" but "is this a cautious agent with demonstrated accuracy on security tasks above threshold X."

---

## Cross-Domain Survey

### LLM Frameworks

| Framework | Role Model | Capability Model | Discovery |
|-----------|-----------|-----------------|-----------|
| **CrewAI** | `role` (free text title), `goal`, `backstory` | `tools` list | None — crew defined in YAML at deploy time |
| **MetaGPT** | `profile` (type enum), `goal`, `constraints` | `actions` list, `react_mode` enum | None |
| **AutoGen** | System prompt (unstructured) | Tool list | None |
| **LangGraph** | Implicit in graph topology | Node function | None — graph statically defined |

**Key finding:** role is expressed as free-form English in a system prompt. Tool list is the closest thing to formal capability. No framework has a discovery mechanism, schema, or registration protocol.

### Game AI

**StarCraft II / SMAC**  
Unit identity is taxonomic: type (worker / fighter / healer / siege) determines attack range, movement, HP. Team composition is an input to the scenario. Role emerges from type attributes, not a named role field.

**AlphaStar training league**  
Three explicit training roles: `main_agent` (maximise win rate), `main_exploiter` (expose flaws), `league_exploiter` (diversify strategies). Plus: race specialisation (closed vocabulary, 3 values) and strategy latent conditioned on human opening moves. Strategy diversity correlates with training success — motivates a `behavioural_niche` dimension absent from all LLM frameworks.

**D&D / RPG party roles**  
Closed vocabulary converged across the industry: Defender (Tank), Striker (DPS), Leader (Healer/Buffer), Controller (AoE/Crowd Control). Two axes: (a) target — self / allies / enemies; (b) mode — absorb / restore / damage / constrain.

**D&D alignment**  
Two-axis dispositional model:
- Order axis: Lawful (rule-following, hierarchical) → Neutral → Chaotic (autonomous, conscience-driven)
- Other-weighting axis: Good (altruistic) → Neutral → Evil (self-interested)

Simple, widely understood, captures meaningful behavioural variance. No analogue in any LLM framework or emerging standard. Directly maps to the "ruthless vs. kind" game example.

**Key finding:** game AI has the richest vocabulary for *what kind of thing an agent is* — functional slot (party role), disposition (alignment), and training role. None of this has crossed into LLM frameworks.

### Academic Multi-Agent Systems (MAS)

**FIPA** (1990s–2000s, now dormant but architecturally sound)  
- AMS (white pages): name → address  
- DF (yellow pages): service type → agents offering it  
- ServiceDescription fields: `name`, `type`, `protocols`, `ontologies`, `languages`, `properties`  
- Discovery via FIPA-ACL `search` to the DF, federated across DF nodes  
- Key insight: separating white pages (identity) from yellow pages (capability) — reappears in A2A

**BDI (Belief-Desire-Intention)**  
Decomposes internal agent state: beliefs (information), desires (motivation), intentions (commitment). Rich vocabulary for internal state; says nothing about external advertisement. A new OWL ontology for BDI aligned with DOLCE exists (Nuzzolese et al., arXiv:2511.17162).

**AgentO** (ESWC 2026, brand new)  
Standardised ontology for agentic AI: agents, tasks, workflows, resource dependencies. Built from 66 agentic workflows across four frameworks. Use cases: declarative reconstruction, cross-context task reuse, workflow auditing. Most directly relevant academic work to this problem.

### Robotics / HRI

**Feature vs. capability distinction** (important, underused in LLM space)  
A robot with a camera has the *feature* `visual_sensor`. The *capability* `object_recognition` requires both the camera and installed software. Capability is context-sensitive — a robot may have it in principle but not in a specific environment.  
Applied to LLMs: an agent may have a tool registered but be rate-limited, context-exhausted, or epistemically unqualified for the current task. Runtime capability availability is unsolved across all LLM frameworks.

**OWL robot capability ontology** (Dussard et al., arXiv:2306.07569)  
Component → Capability inference. Top-level classes: Communication, Mobility, Manipulation, Sensing.

---

## Current Ecosystem Landscape (2025–2026)

### Protocol Layer — Moving Fast

**A2A (Google → Linux Foundation, April 2025)**  
Agent Cards published at `/.well-known/agent-card.json`:
- Identity: `name`, `description`, `provider`, `version`, `url`
- Capabilities: `skills[]` — each with `id`, `name`, `description`, `inputModes`, `outputModes`, `tags`
- Auth: OAuth 2.0, OIDC, mTLS
- Signed Agent Cards (v1.0) for cryptographic provenance

150+ supporting orgs (Google, Microsoft, AWS, Salesforce, SAP, IBM, Workday, ServiceNow). Now the de facto enterprise standard for agent capability advertisement.

**Gap in A2A:** skills describe *what* an agent can do. No functional role slot, no disposition, no behavioural characteristics. Registry and federation explicitly left unspecified — the community is actively debating it ([Discussion #741](https://github.com/a2aproject/A2A/discussions/741)).

**MCP (Anthropic → Linux Foundation, November 2024)**  
Tool access protocol, not agent description. Exposes: tools (name, description, JSON Schema input), resources, prompts. Describes the tools an agent *uses*, not what the agent *is*. 13,000+ servers on GitHub; 20M+ weekly SDK downloads.

**LDP — LLM Delegate Protocol (arXiv:2603.08852, March 2026)**  
Explicitly calls out A2A and MCP's gap: model-level properties are not first-class. Adds: model family, quality hints, reasoning profiles. Closer — but still functional, not dispositional.

**ANP — Agent Network Protocol**  
W3C DIDs + JSON-LD for decentralised agent identity. Agents publish metadata without central registries. Focused on cryptographic identity verification, not capability description.

### Identity/Auth Layer — Active But Incomplete

**OIDC-A (OpenID Connect for Agents, April 2025 proposal)**  
Extends OIDC Core for agents: agent identity claims, delegation chains, attestation. Defines: Agent, Agent Provider, Agent Model, Agent Instance, Delegation Chain. Addresses *who authorised the agent*, not *what kind of agent it is*.

**CoSAI Workstream 4 (March 2026)**  
Agentic Identity and Access Management architectural principles. At the "define the problem" stage. NIST listening sessions began April 2026.

**Key finding (widely cited):** [Resilient Cyber](https://www.resilientcyber.io/p/identity-is-the-agentic-ai-problem): *"Identity is the agentic AI problem nobody has solved yet."* Agents are not humans, not service accounts — they are non-deterministic by design.

### Registry/Discovery Layer — The Most Open Gap

**a2a-registry.org**  
Community registry. Skill/tag-based discovery. AI agents can call it directly for autonomous orchestration. Uses cryptographic identity (secp256k1, DIDs). Computes trust score (0–100) from verification, uptime, transaction history, security scans. No multi-dimensional description.

**W3C WoT Thing Description Directory (TDD)**  
Most mature discovery specification in any domain. SPARQL/JSONPath search over Thing Descriptions. Two-phase: Introduction (DNS-SD, Bluetooth, QR) → Exploration (authenticated TDD query). Designed for IoT but architecturally the right model.

**FIPA DF yellow pages**  
Still the cleanest solved example: register service type → agents offering it → query by criteria. Dormant but correct.

---

## The Gap

| Layer | Who's Working On It | Status |
|-------|---------------------|--------|
| Agent communication protocol | A2A, MCP, ACP, ANP | Converging fast |
| Agent capability advertisement | A2A skills, MCP tools | Partial — functional only |
| Agent authentication/delegation | OIDC-A, CoSAI | Early stage |
| Agent registry/discovery | a2a-registry.org, WoT TDD | Fragmented, no standard |
| **Multi-dimensional agent description** | AgentO (academic only) | **Wide open** |
| **Behavioural disposition** | Nobody | **Completely absent** |
| **Description backed by trust evidence** | Nobody | **Completely absent** |

The behavioural/dispositional axis — *how* an agent approaches tasks, not just *what* tasks it can do — has strong game AI and academic MAS precedent but is entirely absent from LLM frameworks, A2A, MCP, and all current standards bodies.

Connecting self-declared description to *evidence* from actual behaviour (attestations, trust scores) does not exist anywhere.

---

## Proposed Dimension Model

A lightweight `AgentDescriptor` — four layers, building on what A2A and casehub already have:

### Layer 1 — Identity
```
agentId     : URI or structured name (already: actorId)
name        : human-readable
version     : semver
provider    : organisation
```
*Relation to casehub:* `actorId` already covers this. Needs formalisation.

### Layer 2 — Functional Role (the "what slot")
Closed vocabulary, ~6 values:
```
slot: orchestrator | executor | critic | monitor | synthesiser | specialist
```
Game AI analogue: Defender / Striker / Leader / Controller.  
GasTown analogue: Mayor / Polecat / Witness / Deacon / Refinery.

### Layer 3 — Capabilities (the "can do")
```
skills    : list of {id, name, description, inputTypes, outputTypes, tags}
tools     : list of MCP tool references
protocols : list of supported interaction protocols
```
*Relation to casehub:* `CapabilityTag` already scopes trust scores. Skills formalise what those tags mean.

### Layer 4 — Behavioural Disposition (the "how")
```
autonomy       : full | supervised | human_in_loop
delegation     : boolean (can spawn sub-agents)
socialOrient   : altruist | prosocial | individualist | competitive   (SVO axis)
ruleFollowing  : rigid | flexible | autonomous                        (Conscientiousness axis)
riskAppetite   : conservative | balanced | aggressive                 (optional — not yet formalised)
```

**Social orientation (SVO)** replaces the earlier D&D-inspired `otherWeight`. Social Value Orientation is the best-empirically-grounded disposition axis in the literature — validated in behavioural economics, multi-agent RL, and now LLMs (2025–2026). Four-value closed vocabulary. Observable from allocation and cooperation decisions. "Ruthless" = competitive; "kind" = prosocial/altruist.

**Rule-following (Conscientiousness axis)** replaces `orderPref` (lawful→chaotic). Predictive of cooperation and consistency in LLMs. Steerable via representation engineering. Observable from rule adherence, planning behaviour, recovery from ambiguity.

**Risk appetite** keeps emerging from RL exploiter roles and behavioural economics but lacks standard vocabulary. Practically critical for routing decisions (conservative agent for high-stakes tasks; aggressive for open-ended exploration) but not yet formalised enough for a closed vocabulary.

**D&D alignment is dropped.** Intuitively appealing but no academic validation; largely duplicates SVO (good–evil ≈ prosocial–competitive) and Conscientiousness (law–chaos ≈ rigid–autonomous). Not a good foundation.

**Critical finding — behavioural consistency is weak.** Agents exhibit *disposition as response to context* rather than stable traits. Self-declared disposition diverges from actual behaviour across contexts. This makes the attestation layer essential, not optional — you cannot trust a declared SVO profile; peer attestation over time is the only way to validate it. Directly explains MAST FM-2.6 (reasoning-action mismatch, 13.2%): declared disposition is unreliable without evidence.

*All disposition fields are self-declared at registration.* Attestations and trust scores are the *evidence layer* that validates or challenges them — the unique casehub contribution.

---

## The casehub Opportunity

The combination that does not exist anywhere:

```
AgentDescriptor (self-declared)
    + CapabilityTag-scoped trust scores (demonstrated evidence)
    + Attestations (peer verdicts on claims)
    + Registry/discovery query ("find cautious orchestrators with trust > 0.8 in code-review")
```

A2A tells you what an agent *can* do. Nobody connects that to what an agent *has demonstrated* doing, or what kind of agent it *is* behaviourally.

The trust score machinery in casehub-ledger is exactly the evidence layer. The gap is the descriptor and the discovery query.

---

## Agent Card Intellectual Lineage

The A2A Agent Card (Google, April 2025) is the current focal point for agent capability advertisement, but the idea has a clear lineage worth understanding before designing anything:

| Source | Era | Contribution | Gap |
|--------|-----|-------------|-----|
| **FIPA DF (yellow pages)** | 1990s | Agents register service descriptions; others query by capability. The conceptual origin. | Dormant; no adoption in LLM space |
| **W3C WoT Thing Description** | 2017–present | IoT devices self-publish structured JSON-LD at a well-known URL. A2A adopted this pattern directly. More mature than A2A — has SPARQL discovery, federation, the full stack. | Domain-specific (IoT); no agent behavioural model |
| **OpenAPI specs** | 2010s | Operation-level capability description (name, description, input/output schema). A2A Agent Card skills are structurally identical to OpenAPI operations. | Describes APIs, not agents |
| **DNS** | 1980s | Human-readable names resolved to addresses. The ANS (Agent Naming Service) pattern that Solo.io and OWASP both propose is DNS for agents. | Infrastructure only; no semantic content |
| **A2A Agent Card** | April 2025 | Synthesises the above: `/.well-known/agent-card.json` (WoT pattern), skills catalog (OpenAPI pattern), signed cards (PKI), discovery via registry (FIPA DF pattern). | Registry/federation explicitly unspecified; no functional role, disposition, or trust evidence |

### What the Infrastructure Layer Looks Like Now

**Solo.io** identifies three missing pieces A2A intentionally omits:
- **Agent Registry** — central catalog with governance/approval
- **Agent Naming Service (ANS)** — semantic/vector search across capabilities (understands "foreign exchange" → surfaces "forex"/"FX" agents)
- **Agent Gateway** — name resolution, security enforcement, observability, load balancing

Source: [Solo.io blog](https://www.solo.io/blog/agent-discovery-naming-and-resolution---the-missing-pieces-to-a2a) / [agentgateway.dev](https://agentgateway.dev/)

**OWASP ANS (v1.0)** — security-focused naming service. Encodes four dimensions in a structured name: protocol (A2A/MCP/ACP), capability, provider, version. Uses PKI + Zero-Knowledge Proofs for capability validation without exposing sensitive details. DNS for agents with anti-poisoning protection.

Source: [OWASP ANS](https://genai.owasp.org/resource/agent-name-service-ans-for-secure-al-agent-discovery-v1-0/)

### Where Our Angle Differs

Solo.io and OWASP are both building **infrastructure** (registry, naming, gateway, security). They take the Agent Card format as given and add the plumbing around it.

We are asking what **goes on the card** — what dimensions beyond skills/tools/auth should an agent self-declare, and how does actual observed behaviour (attestations, trust scores) validate or challenge those declarations.

These are complementary. An `AgentDescriptor` in casehub could serialise as an A2A Agent Card extension — interoperable with the ecosystem, while adding the functional role, disposition, and trust evidence dimensions that A2A intentionally leaves out.

---

## casehub Vocabulary Alignment

> **Moved to separate doc.** The detailed taxonomy comparison between AgentO and casehub lives in [`casehub-platform-vocabulary-validation.md`](casehub-platform-vocabulary-validation.md). AgentO describes the *parts of an agentic system* (topology, workflows, coordination) — it is out of scope for per-agent description and discovery. The vocabulary validation is useful context for how casehub fits into the broader ecosystem, but it does not inform the `AgentDescriptor` design.

**Summary findings** (details in the separate doc):
- AgentO `Task` ≈ casehub `Work` (not casehub `Task`, which is sub-step) — level mismatch, not a collision
- AgentO `Goal` maps to casehub `Goal` — casehub's definition is stronger (evaluable expression, not desired state string)
- AgentO `Objective` maps to casehub `Case` — the overarching thing a team pursues
- casehub's 4-layer normative model (speech acts, Commitment, Watchdog, CDI enforcement) far exceeds what AgentO has — contribution direction, not import

### Vocabulary rules for `AgentDescriptor`

For the `AgentDescriptor` design — what goes on the card for a single LLM agent:

- Avoid `Task` at this level — use `Work` or `Capability` depending on context
- `Goal`, `Team`, `Resource`, `Environment` — clean adoptions, no conflicts
- Add: functional role slot, disposition dimensions, trust/attestation model — nothing in the ecosystem has these yet

---

## Implementation Philosophy: Ontologies as Design Tools, Not Runtime Dependencies

**The heavy semantic web stack (Jena, RDF4J, OWL API, JADE, Jason) does not belong in casehub.**

These are triplestore and reasoning engines — 150MB+ of infrastructure for storing and querying RDF graphs. casehub-ledger is a Quarkus extension that must be lightweight and zero-cost when idle. Pulling in a triplestore to model a handful of fields on an agent descriptor is the wrong tradeoff.

The distinction:

| Use | Right approach |
|-----|---------------|
| **Design** — what concepts to model, what to call them, how they relate | Read the ontologies (DOLCE, PROV-O, AgentO, OWL-S). Use them as a vocabulary reference and structural guide. Contribute extensions back. Ship none of it. |
| **Runtime** — the `AgentDescriptor` type in a platform library | Plain Java records. Well-chosen field names that align with ontological concepts. Zero deps beyond what Quarkus already has. |
| **Serialisation** — making descriptors interoperable | JSON-LD with an inline `@context`. Jackson is already in Quarkus. The context is a string constant. |
| **Discovery / registry** — SPARQL queries over descriptors | Separate optional module. Consumers who need it add it. Not in the platform API. Same pattern as casehub-ledger's reactive tier: opt-in, build-time gated. |

**The concrete rule:** take the vocabulary from the ontologies; leave the infrastructure behind. AgentO tells us what to call things and how they relate — `Capability`, `Goal`, `Tool`, `Team`, `Constraint`. It does not tell us to embed a Jena triplestore. The JSON-LD `@context` is what links the record fields to the shared ontological vocabulary at wire level — no reasoner required.

**When heavier infrastructure becomes justified:**  
Only if a specific consumer needs inference (e.g., "find all agents whose declared capabilities satisfy constraint X via OWL reasoning") or federated SPARQL queries across multiple registries. That's an optional module in a consuming repo, not a platform library concern.

---

## Open Questions

1. **Where does the descriptor live?** `casehub-platform-api` type? New `casehub-agent-registry` module? Both?

2. **How open is the vocabulary?** `slot` and `orderPref`/`otherWeight` should be closed (controlled vocabulary enables querying). `skills` and `tags` should be open. Where exactly is the boundary?

3. **Static vs. dynamic capability.** Robotics distinguishes feature (hardware present) from capability (enabled in context). Do we need runtime capability availability — e.g., an agent that is rate-limited or context-exhausted? Or is that too operational for a descriptor?

4. **Registration vs. self-declaration.** Does the agent register itself, or does the deployer register it? GasTown: deployer defines roles in config. A2A: agent self-publishes. FIPA: agent registers with DF. The trust model differs per approach.

5. **Relation to A2A Agent Cards.** Should `AgentDescriptor` be an extension of an A2A Agent Card (interoperability with the ecosystem) or independent (no external dependency)?

6. **Disposition attestability.** Self-declared `orderPref: cautious` is just a claim. Can attestors attest to disposition as well as capability? What would that look like?

7. **Discovery query model.** SPARQL (expressive, complex)? JSONPath (simple, limited)? Natural language + embedding similarity? The WoT TDD uses SPARQL. a2a-registry.org uses tag matching + AI routing.

8. **Federation.** If DevTown, GasTown, and Claudony each have registries, how do they federate? A2A Discussion #741 is exploring registry-of-registries with Raft consensus and OpenID Federation.

---

## Citation Trail Findings

### Convergence Point: DOLCE → PROV-O → AgentO

Every paper in the citation trail aligns on the same ontological lineage:

**DOLCE (Descriptive Ontology for Linguistic and Cognitive Engineering)**  
The foundational upper ontology. All agent/capability papers ground their concepts here. DOLCE-Ultralite (OWL form, developed at Italy's Lab for Applied Ontology) is the concrete artefact. The key concepts: objects, events, agents, actions — defined with formal semantics that make reasoning possible. All five starting papers align here.

**PROV-O (W3C PROV Ontology — W3C Recommendation)**  
Describes provenance: Agent → Activity → Entity. Captures responsibility, action attribution, audit trails. Already used by casehub-ledger via `LedgerProvExportService` / `LedgerProvSerializer` for W3C PROV-DM JSON-LD export. This is not coincidence — the provenance layer the research community identifies as critical is already built. Direct reuse opportunity.

**AgentO (ESWC 2026, MIT licence) — system topology vocabulary, not agent description**  
> ⚠️ **Scope note:** AgentO describes the *parts of an agentic system* — architecture, coordination patterns, workflow structure. It is **not** a vocabulary for describing individual LLM agents or making them discoverable. It is out of scope for the `AgentDescriptor` design problem. Retained here as context only — useful if we ever want to formally describe what casehub *is* as a system.

OWL/RDF ontology for agentic AI. Built from 66 real agentic workflows across AutoGen, MetaGPT, CAMEL, CrewAI. 15 core classes:

| Class | What it models |
|-------|---------------|
| `Agent` | The agent entity |
| `Capability` | What the agent can do |
| `Goal` / `Objective` | What it's trying to achieve |
| `Task` / `WorkflowStep` | Units of work |
| `Tool` / `Resource` | What it uses |
| `Memory` / `KnowledgeBase` | What it knows |
| `Team` | Multi-agent composition |
| `Constraint` | Limits on behaviour |
| `Environment` | Operating context |
| `LanguageModel` | The LLM backing |
| `WorkflowPattern` | Reusable workflow structures |

Public SPARQL endpoint: `https://w3id.org/agentic-ai/sparql`  
GitHub: https://github.com/agentic-patterns/agentic-ai-onto  

**Current state:** 12 commits, 1 star, 0 forks. Solid ontological foundation, negligible community. MIT licence means it's a direct adoption target — and low adoption means there's genuine room to shape it through contribution.

**What AgentO is missing** (from our perspective):
- Functional role slot (orchestrator / executor / critic / monitor)
- Behavioural disposition dimensions
- Trust / attestation concepts
- Identity versioning model

These are exactly the dimensions we've identified as the gap in the ecosystem. Contributing them upstream to AgentO is a concrete path.

---

### Supporting Ontologies Worth Reusing

**OWL-S** (W3C submission) — semantic markup for web services. Describes capabilities as ServiceProfile (what it does), ServiceModel (how it works), ServiceGrounding (protocol binding). Mature; cited across all capability papers. Maps cleanly to AgentO's `Capability` class.

**OASIS W3C Community Group** — Ontology for Agents, Systems, and Integration of Services. OWL 2 formalisation, versions 1.0 and 2 available. Active CG; formal standards alignment opportunity. Entry: https://www.w3.org/community/oasis/oasis-version-2/

**WebAgents CG** — W3C Community Group on agent interoperability and autonomy on the web. Convergence with WoT Thing Description underway.

---

### JVM Library Landscape

The semantic web / agent ecosystem has mature JVM libraries. No greenfield needed:

#### Semantic Web / Ontology

| Library | What it does | Status | Key link |
|---------|-------------|--------|---------|
| **Apache Jena** | RDF/OWL graph management, SPARQL, inference | Production | [jena.apache.org](https://jena.apache.org/) |
| **Eclipse RDF4J** | Triplestore, SPARQL endpoints, LMDB/Lucene backends | Production (v5.3.0, Apr 2026) | [rdf4j.org](https://rdf4j.org/) |
| **OWL API** | Parse/manipulate OWL, reasoner integration | Production | [github.com/owlcs/owlapi](https://github.com/owlcs/owlapi) |
| **ELK Reasoner** | OWL 2 EL reasoning (polynomial time, handles large ontologies) | Production | Protégé plugin |
| **Pellet** | OWL DL reasoning + SWRL rules | Production | via OWL API |

#### W3C WoT Thing Description

| Library | What it does | Status |
|---------|-------------|--------|
| **wot-td-java** (Interactions-HSG) | Fluent API for constructing TDs; HTTP/CoAP request execution | Stable — [GitHub](https://github.com/Interactions-HSG/wot-td-java) |
| **wot-jtd** (OEG-UPM) | ORM for Thing Descriptions, SHACL validation | Stable — [GitHub](https://github.com/oeg-upm/wot-jtd) |
| **wot-servient** (sane-city) | Full W3C WoT architecture in Java | Available — [GitHub](https://github.com/sane-city/wot-servient) |
| **Thingweb Directory** | TD directory service with SPARQL query | Available |

WoT Thing Description 2.0 first public working draft published November 2025. No Quarkus extension exists — gap worth filling.

#### Agent Frameworks

| Library | What it does | Status |
|---------|-------------|--------|
| **JADE** | FIPA-compliant Java agent platform; AMS/DF white/yellow pages built in | Production — [jade.tilab.com](https://jade.tilab.com/) |
| **Jason** | BDI AgentSpeak interpreter in Java; LGPL | Production — [jason-lang.github.io](https://jason-lang.github.io/) |

---

### JSON-LD / Schema.org for Web-Scale Discovery

Schema.org + JSON-LD is the de facto standard for describing entities on the web. In 2025–2026, AI crawlers (ChatGPT, Perplexity, Bing Copilot) actively parse `<script type="application/ld+json">` tags. 47.6% of top 10M websites now include JSON-LD.

For agent discovery this matters: an `AgentDescriptor` serialised as JSON-LD with schema.org vocabulary is immediately legible to web-scale AI without any proprietary registry. The `Action` type + `agent` property (type `SoftwareApplication`) is the current pattern.

Mapping path: `AgentDescriptor` → AgentO OWL → JSON-LD serialisation → schema.org compatibility layer → A2A Agent Card extension.

---

### Contribution Opportunities

| Target | What to contribute | Entry point |
|--------|-------------------|-------------|
| **AgentO** | Functional role slot, disposition dimensions, trust/attestation concepts, identity versioning | [github.com/agentic-patterns/agentic-ai-onto](https://github.com/agentic-patterns/agentic-ai-onto) |
| **OASIS W3C CG** | LLM-specific agent extensions to the OWL 2 formalisation | [w3.org/community/oasis](https://www.w3.org/community/oasis/oasis-version-2/) |
| **WebAgents CG** | Agent trust and attestation model | W3C CG directory |
| **wot-td-java or Quarkus** | Quarkus extension for WoT Thing Description | No extension exists |
| **A2A ecosystem** | Extended Agent Card schema with disposition/role dimensions | [a2aproject/A2A](https://github.com/a2aproject/A2A) |

---

## LDP Analysis — Most Directly Relevant Prior Work

**LDP (LLM Delegate Protocol, arXiv:2603.08852, March 2026)** is the closest existing work to what we're designing. Full schema documented here for reference.

### Delegate Identity Card — full schema (Appendix B)

**Core identity:**
| Field | Type | Purpose |
|-------|------|---------|
| `delegate_id` | string | Unique identifier |
| `principal_id` | string | Owning organisation |
| `model_family` | string | Model lineage (e.g. "qwen") |
| `model_name` | string | Specific model |
| `model_version` | string | Version tag |
| `runtime_version` | string | Inference runtime |
| `weights_fingerprint` | string | Weight integrity hash |
| `endpoint_address` | string | Network location |

**Trust / security:**
| Field | Type | Purpose |
|-------|------|---------|
| `trust_domain` | string | Security boundary label |
| `public_key` | string | Cryptographic identity |
| `jurisdiction` | string | Regulatory scope |
| `data_handling_policy` | string | Data governance rules |

**Capabilities** (per-capability object array):
| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Skill label |
| `quality_hint` | float 0–1 | Continuous quality score |
| `latency_hint_ms_p50` | integer ms | Median latency estimate |
| `cost_hint` | low/medium/high | Cost tier |
| `context_window` | integer | Max token context |
| `modalities_supported` | string[] | e.g. ["text"] |
| `languages_supported` | string[] | e.g. ["en", "zh"] |

**Behavioural:**
| Field | Type | Purpose |
|-------|------|---------|
| `reasoning_profile` | string | Freeform style — "deep-analytical", "fast-practical" |
| `cost_profile` | low/medium/high | Overall cost tier |

**Provenance on every task result:**
```json
{
  "produced_by": "delegate:qwen3-8b",
  "model_version": "qwen3-8b-2026.01",
  "confidence": {"score": 0.84, "method": "self-report"},
  "verification": {"performed": true, "status": "passed"}
}
```

### Key empirical findings

- Identity-aware routing: ~12× lower latency on easy tasks (routes cheap model, not expensive one)
- Semantic frame payload: 37% token reduction, no quality loss
- **Critical:** noisy provenance (inflated confidence, false verification) "degrades quality *below* the no-provenance baseline" — self-declared claims without verification are actively harmful

### Where LDP lands relative to our design

| LDP concept | casehub equivalent | Verdict |
|-------------|-------------------|---------|
| `weights_fingerprint` | `agentConfigHash` in `ProvenanceSupplement` | Already built — convergent design |
| `quality_hint` per capability | `ActorTrustScore` per `CapabilityTag` | casehub's version is evidence-backed; LDP's is designer-assigned |
| `trust_domain` | Tenant/deployment scoping | Not yet modelled in `AgentDescriptor` |
| `jurisdiction` + `data_handling_policy` | Not in casehub yet | New dimension worth adding |
| `latency_hint_ms_p50` | Not in casehub | New operational dimension |
| `cost_profile` | Not in casehub | New routing dimension |
| `reasoning_profile` (freeform string) | Our two-axis disposition model | LDP's is one-dimensional and freeform; structured axes are stronger |
| `verification.performed/status` | Attestation verdict (SOUND/FLAGGED/ENDORSED/CHALLENGED) | casehub's attestation model is richer |

**The noisy provenance finding is the empirical case for casehub's architecture.** LDP shows that unverified self-declared hints make things *worse* than no metadata at all. The fix is exactly what casehub-ledger provides: peer attestation replacing designer-assigned hints.

**`reasoning_profile` is LDP's stab at disposition** — one dimension, freeform string. Our two-axis model (orderPref × otherWeight) is more principled and queryable.

---

## MAST Analysis — Empirical Failure Evidence

**MAST: "Why Do Multi-Agent LLM Systems Fail?"**  
arXiv:2503.13657 · NeurIPS 2025 Spotlight · [GitHub](https://github.com/multi-agent-systems-failure-taxonomy/MAST)  
1,642 annotated traces across AutoGen, CrewAI, LangGraph. 14 failure modes, 3 categories.

Overall failure rate: **41–86.7%** across 7 production MAS frameworks.

### The 14 failure modes

**FC1 — System Design Issues**  
FM-1.2: Disobey role specification → explicitly cascades into FC2 misalignment

**FC2 — Inter-Agent Misalignment (36.9% of all failures)**

| Mode | Rate | What it is |
|------|------|-----------|
| FM-2.6 | 13.2% | Reasoning-action mismatch — agent states it will do X, does Y |
| FM-2.3 | 7.4% | Task derailment — drifts from original objective |
| FM-2.2 | 6.8% | Fails to ask for clarification — proceeds on wrong assumptions |
| FM-2.1 | 2.2% | Conversation reset — loses prior context |
| FM-2.5 | 1.9% | Ignores other agents' input |
| FM-2.4 | 0.85% | Information withholding |

**FC3 — Task Verification & Termination (21.3%)**

### What MAST means for agent description

**FM-2.6 (13.2%) is the key finding.** Reasoning-action mismatch — an agent whose declared `reasoning_profile` doesn't match actual behaviour. This is precisely what peer attestation catches over time: FLAGGED verdicts accumulate when an agent consistently says one thing and does another. The trust score for that capability degrades accordingly. What LDP would route around by choosing a different `reasoning_profile` label, casehub-ledger makes observable and persistent.

**FM-1.2 → FC2 cascade is the empirical argument for a formal role slot.** Poorly defined roles feed misalignment downstream. Freeform system prompts as role definitions are insufficient — the data shows it. A structured `slot` field (orchestrator / executor / critic / monitor) is failure prevention, not just routing convenience.

**"Deference to Expertise" (their HRO framing).** The authors frame good MAS as organisational design — FM-2.2 (fails to ask for clarification) undermines *deference to expertise*, the HRO principle of knowing when to yield to others' knowledge. This is exactly what capability-aware routing does: an agent that knows its peer's trust score per capability knows when to defer rather than proceed on wrong assumptions.

### Their recommended fixes and casehub's structural answers

| MAST recommendation | casehub equivalent |
|--------------------|-------------------|
| Probabilistic confidence thresholds | `TrustGateService.meetsThreshold()` |
| Modular, single-responsibility agents | Functional role slot in `AgentDescriptor` |
| Standardised communication protocols | A2A Agent Cards + Qhorus speech act taxonomy |
| Cross-verification between agents | Peer attestation (SOUND/FLAGGED/ENDORSED/CHALLENGED) |
| Enforce hierarchy / role authority | `AgentDescriptor.slot` + delegation flag |

**What MAST doesn't have:** no formal capability declaration, no disposition model, no trust evidence layer connecting self-description to demonstrated behaviour. Their fixes are prompt-based and reactive. The structural solution is what we're building.

**Dataset:** MAST-Data (1,642 traces) is publicly available. Potential validation resource if we want to measure whether typed role/disposition descriptions reduce FC2 failure rates.

---

## Disposition Research — What Six Threads Found

### What converges

Two axes recur independently across behavioural economics, game AI, HRI, personality science, and LLM research:

**Axis 1 — Social Value Orientation (SVO)**  
The self-vs.-other tradeoff. Closed vocabulary: Altruist → Prosocial → Individualist → Competitive. Validated in: behavioural economics (SVO slider task), multi-agent RL (arXiv:2603.13890), and LLMs directly (arXiv:2605.14034, 2502.12504). Observable from allocation, cooperation, and negotiation decisions. The strongest empirically-grounded disposition axis available.

**Axis 2 — Conscientiousness / Rule-following**  
The constraint-vs.-autonomy tradeoff. Maps to Big Five Conscientiousness + inverse Openness. Three-value vocabulary: rigid → flexible → autonomous. Predictive of cooperation and consistency in LLMs. Steerable via representation engineering (arXiv:2503.12722 — α=±3.5 steer without fluency degradation). Observable from rule adherence, planning sophistication, recovery from ambiguity.

**Axis 3 — Risk Appetite** (emerging, not yet formalised)  
Appears in RL exploiter roles (AlphaStar: main vs. exploiter), behavioural finance, and uncertainty tolerance research. Three-value vocabulary: conservative → balanced → aggressive. Practically important for routing but no standard measurement framework yet.

### Why D&D alignment is dropped

The law–chaos × good–evil axes are intuitively appealing but:
- No peer-reviewed empirical validation for AI agents
- Good–evil ≈ SVO (altruist–competitive) — conceptual duplication
- Law–chaos ≈ Conscientiousness + inverse Openness — conceptual duplication
- No observable measurement operationalisation

SVO + Conscientiousness cover the same space with actual empirical grounding.

### The key architectural finding — consistency is weak

**Agents exhibit disposition as response to context, not as stable trait.** Questionnaire-based measurement (psychometric) diverges from behavioural measurement across contexts. Self-declared SVO does not reliably predict actual allocation behaviour in novel contexts (arXiv:2602.01063, 2604.28048).

This is the empirical argument for the attestation layer. An `AgentDescriptor` with self-declared `socialOrient: prosocial` is a claim, not a fact. Peer attestation over time — verdicts on actual cooperation decisions — is the only way to validate or challenge it. casehub-ledger's attestation machinery is the right tool; no other framework has it.

**The personality subnetworks finding** (arXiv:2602.07164) adds nuance: LLMs already contain personality-like latent structures. Disposition isn't entirely declared — it's emergent and detectable from behaviour. Attestors observe actual decisions; the declared `socialOrient` is a starting prior that evidence updates.

### Key references

- [Nature MI 2025](https://www.nature.com/articles/s42256-025-01115-6) — psychometric framework for LLM personality evaluation
- [arXiv:2603.13890](https://arxiv.org/abs/2603.13890) — Beyond Self-Interest: SVO in multi-agent LLMs
- [arXiv:2605.14034](https://arxiv.org/abs/2605.14034) — From Descriptive to Prescriptive: SVO alignment
- [arXiv:2503.12722](https://arxiv.org/abs/2503.12722) — Conscientiousness steering via representation engineering
- [arXiv:2602.01063](https://arxiv.org/abs/2602.01063) — Personality expression across contexts (consistency is weak)
- [arXiv:2602.07164](https://arxiv.org/abs/2602.07164) — Personality subnetworks in LLMs
- [arXiv:2502.12504](https://arxiv.org/abs/2502.12504) — Simulating prosocial behaviour in multi-agent LLMs

---

## Generative Direction — Descriptor as CLAUDE.md Generator

This emerged from the discovery research but is a distinct direction worth capturing separately.

### The insight

The `AgentDescriptor` is not only a discovery artifact — it is a **generator**. Given a Case Goal and a descriptor configuration, you can render a CLAUDE.md (or any system prompt) that embeds the agent's role, disposition, capabilities, and specific objective in natural language. The ontology becomes a template engine.

### The generation loop

```
CaseDefinition.Goal  ──┐
AgentDescriptor       ──┼──▶  render CLAUDE.md  ──▶  run agent  ──▶  outcomes
  (slot, socialOrient, │                                                  │
   ruleFollowing,      │                                                  ▼
   capabilities, ...)  │                                          attestations
                       │                                                  │
                       └──────────────── update trust scores / knowledge graph
```

1. **Goal** comes from the Case (`CaseDefinition.Goal`) — already a formal expression in casehub-engine. You read it; you don't invent it.
2. **Descriptor** defines how the goal is pursued — role slot, SVO, conscientiousness, capability set.
3. **Rendered CLAUDE.md** is the natural language materialisation: role + disposition + goal + capability context, templated into prose.
4. **Outcomes** feed back via attestations → trust scores per capability per disposition configuration.

### Concrete example — same goal, two descriptors

**Goal:** "Review this PR. Accept only if code quality and test coverage meet the team standard."

**Descriptor A:**
```yaml
slot: critic
socialOrient: prosocial
ruleFollowing: flexible
capabilities: [code-review]
context: "reviewer of intern submissions"
```
**Renders as:**
> "Your goal is to review this PR and accept only if code quality and test coverage meet the team standard. You are a code reviewer working with interns. Be skeptical but kind — your role is coaching, not gatekeeping. Adapt your feedback to the author's level."

**Descriptor B:**
```yaml
slot: critic
socialOrient: competitive
ruleFollowing: rigid
capabilities: [code-review]
context: "gatekeeper for production branch"
```
**Renders as:**
> "Your goal is to review this PR and accept only if code quality and test coverage meet the team standard. You are a ferocious gatekeeper. Standards are non-negotiable. Reject anything that wastes the team's time. Do not soften feedback."

Same goal. Same capability. Radically different behaviour from two structured dimension choices.

### Why this matters

**Systematic experimentation.** Hold the goal constant. Vary the disposition. Observe what changes. This is structured prompt engineering with a controlled variable — something no current framework supports because they lack a formal descriptor schema.

**Institutional memory.** Once you have enough `(descriptor, task-type, outcome, attestation)` tuples in the knowledge graph, you can query it: *"For PR reviews on intern code, which SVO configuration produced the fewest escalations and the best learning outcomes?"* The graph becomes organisational memory for which agent configurations work for which task types.

**Closes the feedback loop the research is missing.** LDP has the descriptor side. MAST has the failure evidence. Neither connects descriptors to outcomes through an attestation layer. That's the unique position.

### Rendering architecture — strategy decision

**Strategic constraint:** current target is CLAUDE.md (Claude Code format). Future target is any LLM system prompt format. The architecture must support both from the start without the descriptor knowing about either.

**Solution: `SystemPromptRenderer` SPI — same pattern as the rest of casehub.**

```
AgentDescriptor + CaseGoal + context
        │
        ▼
SystemPromptRenderer  (SPI in casehub-platform-api)
        │
        ├── ClaudeMarkdownRenderer   (@DefaultBean — current)
        ├── OpenAISystemPromptRenderer
        ├── A2AAgentCardRenderer
        └── [future — Gemini, Llama, etc.]
```

The descriptor is the format-agnostic source of truth. The renderer is the format-specific adapter. Adding a new LLM target = new renderer implementation, nothing in the descriptor or SPI changes.

**Hybrid rendering within each renderer:**

| Layer | Tool | What it handles | Determinism |
|-------|------|----------------|-------------|
| **Template** | Qute (Quarkus native) | Structural skeleton — role header, goal injection, capability list, format-specific sections (CLAUDE.md has specific section structure) | Fully deterministic |
| **LLM prose** | Lightweight LLM call | Disposition section only — renders SVO + conscientiousness + context into 2–3 sentences of natural language behavioural instruction | Cached by (descriptor hash + context hash) |

The LLM-for-prose call happens at **descriptor registration time**, not at agent runtime. Generate once, store, reuse. Regenerate only when the descriptor changes. This keeps runtime behaviour deterministic and keeps the generation cost amortised.

**The CLAUDE.md specifics** live entirely inside `ClaudeMarkdownRenderer`. When rendering for other LLMs, none of that leaks into the descriptor or SPI contract.

**ADR candidate:** this decision has real architectural consequences — where the SPI lives, what the renderer contract looks like, how caching works, which module owns `ClaudeMarkdownRenderer`. Should be formalised before implementation.

### What needs building

| Component | Status | Notes |
|-----------|--------|-------|
| `AgentDescriptor` schema | Design phase | Core platform type — `casehub-platform-api` |
| `SystemPromptRenderer` SPI | Design phase | `casehub-platform-api` — format-agnostic contract |
| `ClaudeMarkdownRenderer` | Not started | `@DefaultBean` — Qute template + LLM prose call |
| `CaseDefinition.Goal` → renderer input | Exists in engine | Read the goal; inject into render call |
| Rendered prompt cache | Not started | Keyed by (descriptor hash + context hash) |
| Outcome → attestation pipeline | Exists | casehub-ledger — already built |
| Knowledge graph of (descriptor, task, outcome) | Not started | Long-game store |

### Open questions

- **What is the rendering unit?** A structured section agents include in their own CLAUDE.md, or a full file? Probably a section — agents may have repo-specific content that must coexist.
- **Which LLM generates the prose?** Any LLM via the platform's configured model. The renderer doesn't hardcode a model.
- **How does context vary rendering?** `"coaching interns"` vs `"production gatekeeper"` changes the prose for the same SVO value. Context is an explicit field in the render call, not inferred.
- **Iteration cadence?** How many runs before you have enough signal to prefer one descriptor configuration for a given task type? Probably domain-specific — fast feedback tasks (PR review) vs. slow (clinical decisions).

---

## Ecosystem Fragmentation Warning

**Observed 2026-05-22 via A2A Discussion #1631.**

The agent trust/reputation problem is attracting proprietary solutions, not a converging standard. Multiple independent implementations are being built in parallel, each anchoring on different technology choices:

| Party | Approach | Technology bet | Risk |
|-------|----------|---------------|------|
| ARP (makito20256) | Append-only transaction ledger, deterministic scoring | Custom "trust substrate" protocol, own URI namespace | Proprietary lock-in |
| MoltBridge (JKHeadley) | Ed25519-signed attestations + graph traversal | Neo4j, "credibility packets" | Platform-specific |
| laplace0x | On-chain identity as trust substrate | ERC-8004, Ethereum | Blockchain dependency |
| Dispute resolution proposals | Escrow-bound dispute layer | Unspecified | Fragmented on top of fragmented |

**None of these are coordinating on a shared standard.** They are all responding to the same Discussion #1631 thread but building separate proprietary solutions. Classic early-market fragmentation — same pattern as pre-HTTP networking protocols, pre-OAuth identity, pre-OpenAPI API description.

**What this means for Eidos:**

1. **The problem is unambiguously real.** People don't spend engineering effort on theoretical problems. The fragmentation confirms the gap.

2. **The standard is not yet settled.** A2A Discussion #1631 is at draft v0.0.3. The window to influence the canonical solution is open — but it closes once one of these gets production deployment and ecosystem lock-in.

3. **casehub's posture is different from every party in that thread.** We are not building a proprietary trust substrate. We are building a platform capability that sits on open standards (A2A extension mechanism, JWS, PROV-O) and contributes back. That's a stronger long-term position.

4. **Strategic question unresolved:** when we engage, do we engage as a casehub-specific implementation, or do we attempt to propose something that becomes the open standard? The latter is harder but more valuable to the ecosystem. Both are viable; the choice shapes how we frame the contribution.

**The anti-gaming gap nobody has solved:** Discussion #1631 identified Sybil attacks (synthetic agent networks rating each other) and trust-building-then-exploit as real threats. None of the proposals have a complete answer. casehub-ledger's bilateral signing + key rotation + SUSPECT detection addresses the identity layer of this — but the social gaming attack (real agents colluding) is still open for everyone.

---

## Areas to Keep Digging

The core research question is: **how do you describe an individual LLM agent's identity, characteristics, and capabilities so it can be discovered and its claims validated by trust evidence?** AgentO is out of scope for this — it describes system topology, not individual agents.

**Per-agent description — the actual problem:**
- [x] LDP paper (arXiv:2603.08852) — done. Schema documented above. Key gap: `reasoning_profile` is one-dimensional freeform; `quality_hint` is designer-assigned not evidence-backed.
- [x] Disposition thread — done. SVO + Conscientiousness replace D&D axes. Consistency weakness validated empirically — attestation layer is essential not optional.
- [ ] A2A Agent Card spec in detail — what exactly goes in a skill object; can it be extended without breaking compatibility?
- [ ] A2A Discussion #741 — registry/federation proposals; where does a casehub registry plug in?
- [ ] OIDC-A proposal — delegation chain model; how does it interact with `delegation` dimension and multi-agent trust chains?
- [ ] FIPA Agent Description (DF) in detail — the cleanest solved example of per-agent capability advertisement for discovery; what can we reuse conceptually?

**Behavioural disposition — the missing axis:**
- [ ] AlphaStar strategy latent / `main_exploiter` / `league_exploiter` roles — how does RL formalise behavioural niche? Does it suggest anything for the disposition model?
- [ ] Is there prior work on *attestable* disposition claims — not just self-declared, but peer-verified over time?
- [ ] MAST failure taxonomy (1,600 traces — 36.9% inter-agent misalignment) — does typed disposition reduce misalignment? Any empirical data?

**Discovery / registry:**
- [ ] WoT Thing Description Directory spec — the most mature discovery implementation; how does SPARQL search work over TD properties? Map to `AgentDescriptor` fields.
- [ ] WoT TD 2.0 first public working draft (November 2025) — what changed from 1.1; is a Quarkus extension worth building?
- [ ] a2a-registry.org approach — how does semantic/vector search work in practice; what query model?

**Trust evidence layer:**
- [ ] Is there prior work connecting self-declared capability claims to attestation/verification? (Not just OIDC token claims — actual peer evidence)
- [ ] CoSAI Workstream 4 — what architectural principles they've settled on for agent trust

---

## References

**Game AI**
- Samvelyan et al. (2019). *The StarCraft Multi-Agent Challenge*. [arXiv:1902.04043](https://arxiv.org/abs/1902.04043)
- Vinyals et al. (2019). *Grandmaster level in StarCraft II*. Nature / [DeepMind](https://deepmind.google/blog/alphastar-grandmaster-level-in-starcraft-ii-using-multi-agent-reinforcement-learning/)

**Academic MAS**
- FIPA Agent Management Specification. [fipa.org](http://www.fipa.org/specs/fipa00023/XC00023H.html)
- Nuzzolese et al. (2025). *The BDI Ontology*. [arXiv:2511.17162](https://arxiv.org/abs/2511.17162)
- Hierarchical MAS Taxonomy (2025). [arXiv:2508.12683](https://arxiv.org/html/2508.12683v1)
- AgentO (ESWC 2026). [Springer](https://link.springer.com/chapter/10.1007/978-3-032-25159-6_16)

**Robotics**
- Dussard et al. (2023). *Ontological robot capability description*. [arXiv:2306.07569](https://arxiv.org/abs/2306.07569)

**LLM Agent Surveys**
- Agentic AI taxonomy (2025). [arXiv:2601.12560](https://arxiv.org/html/2601.12560v1)
- Agent Skills survey (2025). [arXiv:2602.12430](https://arxiv.org/html/2602.12430v3)
- LDP: Identity-Aware Protocol (2026). [arXiv:2603.08852](https://arxiv.org/html/2603.08852v1)

**Emerging Standards**
- A2A Protocol Specification. [a2a-protocol.org](https://a2a-protocol.org/latest/specification/)
- A2A Registry Discussion. [GitHub #741](https://github.com/a2aproject/A2A/discussions/741)
- a2a-registry.org. [a2a-registry.org](https://www.a2a-registry.org/)
- MCP Specification (Nov 2025). [modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-11-25)
- W3C WoT Thing Description 1.1. [w3.org](https://www.w3.org/TR/wot-thing-description11/)
- W3C WoT Discovery. [w3.org](https://www.w3.org/TR/wot-discovery/)
- OIDC-A Proposal. [subramanya.ai](https://subramanya.ai/2025/04/28/oidc-a-proposal/)
- Solo.io on missing A2A pieces. [solo.io](https://www.solo.io/blog/agent-discovery-naming-and-resolution---the-missing-pieces-to-a2a/)
- Identity unsolved. [Resilient Cyber](https://www.resilientcyber.io/p/identity-is-the-agentic-ai-problem)
- CoSAI Workstream 4 / NIST sessions. [secureflo.net](https://secureflo.net/ai-agent-identity-management-a-2026-ciso-playbook/)
