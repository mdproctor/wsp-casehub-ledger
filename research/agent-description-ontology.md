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

## Areas to Keep Digging

- [ ] Read AgentO paper in full (ESWC 2026) — most relevant academic work
- [ ] A2A Discussion #741 — what the community is converging on for registry/federation
- [ ] LDP paper (arXiv:2603.08852) — how they model reasoning profiles and quality hints
- [ ] WoT Thing Description Directory spec — best worked example of semantic discovery
- [ ] BDI ontology (arXiv:2511.17162) — does belief/desire/intention decomposition add anything useful?
- [ ] OIDC-A proposal — how delegation chains are modelled; relevant to `delegation: boolean`
- [ ] CoSAI Workstream 4 — what architectural principles they've settled on
- [ ] Agentic AI taxonomy (arXiv:2601.12560) — six-component model: Perception, Brain, Planning, Action, Tool Use, Collaboration
- [ ] Is there prior work on *attestable* capability claims? (Not just self-declared)

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
