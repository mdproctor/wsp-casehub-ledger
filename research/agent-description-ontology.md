# Agent Description Ontology — Research

**Status:** Active research — not yet a spec  
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
autonomy    : full | supervised | human_in_loop
delegation  : boolean (can spawn sub-agents)
orderPref   : lawful | neutral | chaotic
otherWeight : collaborative | neutral | competitive
```
`orderPref` + `otherWeight` is the D&D alignment model applied to agents.  
Game example: ruthless = competitive + chaotic. Kind = collaborative + lawful.  
Trust example: cautious orchestrator = lawful + supervised; exploratory researcher = chaotic + full autonomy.

*These are self-declared at registration.* Attestations and trust scores become the *evidence layer* that validates or challenges them over time — which is the unique casehub contribution.

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

## AgentO Taxonomy vs. casehub Terminology

A deliberate comparison before any vocabulary decisions. casehub's terminology has been carefully chosen — especially the work/task/goal distinction — and any adoption from AgentO must not reintroduce collisions that were explicitly designed away.

### Term-by-term comparison

| AgentO concept | casehub equivalent | Verdict |
|---|---|---|
| `Task` | `WorkItem` (casehub-work) / `PlanItem` (casehub-engine) / `humanTask` (YAML binding) | **COLLISION — do not import** |
| `WorkflowPattern` | `CaseDefinition` (CNCF Serverless Workflow layer) | **Overlap — different level, watch for drift** |
| `WorkflowStep` | `PlanItem` at execution stage | **Overlap — different level, watch for drift** |
| `Goal` | Not formally defined in casehub | Clean adoption |
| `Objective` | Not formally defined in casehub | Clean adoption |
| `Team` | Not in casehub | Clean adoption |
| `Capability` | `CapabilityTag` (trust scoping in casehub-ledger) | Different levels — coexist safely |
| `Constraint` | `SlaBreachPolicy`, `ExclusionPolicy` (policy/decision objects) | Different semantic level — do not conflate |
| `Memory` / `KnowledgeBase` | Not in casehub | Clean adoption |
| `Environment` | Not in casehub | Clean adoption |
| `Resource` | Not in casehub | Clean adoption |
| `Tool` | MCP tools (casehub-qhorus) | Weak overlap, different scope |
| `LanguageModel` | Implicit in `actorId` model-family | No conflict |
| `Agent` | `actorId` (casehub-ledger), `CurrentPrincipal` (platform) | Different levels — can coexist |

### The `Task` collision is load-bearing

casehub-work explicitly does **not** call WorkItems "Tasks" — the doc notes it is *"deliberately NOT called Task"* because CNCF Serverless Workflow and casehub-engine both use `Task` with distinct, conflicting semantics. Importing AgentO's `Task` vocabulary into any casehub `AgentDescriptor` or platform type would reintroduce exactly the collision that was designed away. **Do not use `Task` in any casehub agent vocabulary.**

### Where casehub is ahead of AgentO — the normative layer

AgentO's normative model: `Constraint` as a subclass of `KnowledgeBase` — stored rules that restrict or guide agent behaviour. This is a knowledge-representation concept, not a deontic model.

casehub-qhorus has a full 4-layer normative accountability framework grounded in Austin/Searle speech act theory:

1. **Illocutionary** — what was said: 9 speech-act types (Directive, Commissive, Assertive, Declarative, Expressive, etc.)
2. **Commitment** — what was obligated: `Commitment` entity with 7-state lifecycle (OPEN → FULFILLED / FAILED / EXPIRED / ...)
3. **Temporal** — when obligations become stale: Watchdog, deadline enforcement
4. **Enforcement** — casehub-engine orchestration reacts to commitment outcomes via CDI events

The 3-channel normative layout (`work` / `observe` / `oversight`) structures agent-to-agent and agent-to-human interactions. This is a genuine deontic model — obligations, commitments, temporal validity — that AgentO does not have in any form.

**This is a contribution opportunity, not an import.** casehub's normative model is what we'd give to AgentO, not take from it.

### Vocabulary rules for `AgentDescriptor`

- **Use from AgentO:** `Goal`, `Objective`, `Team`, `Capability`, `Memory`, `Environment`, `Resource` — no conflicts
- **Do not use:** `Task` — collision with WorkItem/PlanItem distinction
- **Use carefully:** `WorkflowPattern`, `WorkflowStep` — keep scoped to agent-level description; do not let them bleed into casehub-engine's CaseDefinition/PlanItem layer
- **Extend:** Add functional role slot, disposition dimensions, trust/attestation model — not in AgentO

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

**AgentO (ESWC 2026, MIT licence)**  
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

## Areas to Keep Digging

**Ontology / standards:**
- [ ] Read AgentO OWL file in full — map all 15 classes + properties to proposed dimensions; identify what we'd extend vs. add
- [ ] OWL-S ServiceProfile spec — how it models input/output/preconditions; map to AgentO `Capability`
- [ ] OASIS W3C CG version 2 — what's in the OWL 2 formalisation; is it worth contributing to vs. AgentO?
- [ ] BDI ontology (arXiv:2511.17162) — does belief/desire/intention decomposition add anything useful beyond Goal/Objective in AgentO?
- [ ] WoT Thing Description Directory spec — best worked example of semantic discovery; map affordances to agent capabilities
- [ ] WoT TD 2.0 first public working draft (November 2025) — what changed; worth building a Quarkus extension?

**Protocols:**
- [ ] LDP paper (arXiv:2603.08852) in full — how they model reasoning profiles and quality hints; map to disposition dimensions
- [ ] A2A Discussion #741 — what the community is converging on for registry/federation; where to plug in
- [ ] OIDC-A proposal — delegation chain model; relevant to `delegation` dimension and multi-agent trust chains
- [ ] CoSAI Workstream 4 — what architectural principles they've settled on; where trust evidence fits

**Theory / depth:**
- [ ] Agentic AI taxonomy (arXiv:2601.12560) — six-component model (Perception, Brain, Planning, Action, Tool Use, Collaboration); does it map to AgentO classes?
- [ ] Is there prior work on *attestable* capability claims — not just self-declared, but peer-verified?
- [ ] AlphaStar strategy latent / behavioural niche — how to encode diversity as a dimension without over-specifying

**Code / libraries:**
- [ ] Run AgentO Turtle file through Jena — what can be inferred; how hard to embed in a Quarkus app
- [ ] wot-jtd SHACL validation — can it validate an AgentDescriptor shape?
- [ ] JADE DF (yellow pages) source — compare to what we'd need for a casehub registry; is reuse viable or is JADE too heavyweight?

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
