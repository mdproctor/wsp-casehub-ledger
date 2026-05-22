# casehub Platform Vocabulary — External Validation

**Status:** Reference  
**Started:** 2026-05-22  
**Purpose:** Validates casehub's terminology and taxonomy against the AgentO ontology and broader academic MAS vocabulary. Confirms casehub's concepts are well-grounded and positioned for contribution to the ecosystem.

This is separate from the agent description / discovery research (`agent-description-ontology.md`). That research asks "how do we describe individual LLM agents for discovery?" This document asks "does casehub's vocabulary align with established agentic AI concepts?"

---

## AgentO as a Validation Reference

**AgentO (ESWC 2026, MIT)** — an OWL/RDF ontology derived from 66 real agentic workflows across AutoGen, MetaGPT, CAMEL, CrewAI. It describes the *parts of an agentic system*: agents, tasks, workflows, teams, goals, resources. Useful as an external vocabulary reference — not as a design input for per-agent description.

GitHub: https://github.com/agentic-patterns/agentic-ai-onto  
SPARQL endpoint: `https://w3id.org/agentic-ai/sparql`

---

## Vocabulary Mapping

### casehub's formal taxonomy (as defined)

From **ADR-0003** (`engine/adr/0003-work-workitem-task-naming.md`):

| Term | casehub meaning |
|------|----------------|
| **Work** | Generalised assignable unit — top-level concept; automated or human |
| **WorkItem** | Human-inbox specialisation of Work — claim/inbox semantics, 10-status lifecycle |
| **Task** | Sub-steps *within* a Work unit — lowest granularity |

From the engine model (`CaseDefinition`, `CaseInstance`):

| Term | casehub meaning |
|------|----------------|
| **Case** | An ACM instance pursuing one or more Goals — the overarching objective |
| **Goal** | Formal evaluable completion condition on `CaseDefinition`; fires `GoalReached` when satisfied; case → COMPLETED when all goals reached |
| **Milestone** | Intermediate checkpoint within a case; fires `MilestoneReached` |
| **Binding** | Condition-driven dispatch rule within a case |
| **PlanItem** | Unit of work within the case plan |
| **CaseContext** | Append-via-EventLog blackboard — shared agent working state |

### Term-by-term comparison

| AgentO concept | AgentO definition | casehub equivalent | Finding |
|---|---|---|---|
| `Task` | "A specific activity that contributes to achieving a goal" | **Work** (generalised assignable unit) | Level mismatch — AgentO Task ≈ casehub Work, not casehub Task (which is sub-step) |
| `Goal` | "An objective or desired state an agent aims to achieve" | `Goal` (formal evaluable completion condition) | Compatible — casehub is more precise: Goal is machine-evaluable, not a desired state string |
| `Objective` | "A collective objective a team is assigned to accomplish" | **Case** / `CaseDefinition` | Clean mapping — a Case is the objective the agent team pursues |
| `WorkflowPattern` | "Reusable template defining task sequences" | `CaseDefinition` (CNCF Serverless Workflow) | Different name, similar concept at different abstraction level |
| `WorkflowStep` | "Individual action or phase within a workflow pattern" | `Binding` / `PlanItem` | Different name, similar concept |
| `Team` | "Coordinated group of agents pursuing a common objective" | Pool of Workers on a Case | Implied, not explicit — clean adoption if needed |
| `Capability` | "Abilities performable by an agent" | `CapabilityTag` (trust scoping in ledger) | Different levels — AgentO Capability is descriptive; casehub CapabilityTag is an evidence-scoping instrument |
| `Constraint` | Subclass of KnowledgeBase — "rules restricting behaviour" | `SlaBreachPolicy`, `ExclusionPolicy` | Different semantic level — casehub constraints are policy/decision objects, not stored knowledge |
| `Memory` / `KnowledgeBase` | Structured information store | `CaseContext` (the blackboard) | Compatible — CaseContext is the shared agent working memory |
| `Environment` | "Surroundings or context in which agent operates" | `CaseContext` + deployment context | Partial overlap |
| `Resource` | "Any asset utilised by agent to perform tasks" | Not in casehub explicitly | Clean adoption if needed |
| `Tool` | "Instrument used by agent to enhance capabilities" | MCP tools (casehub-qhorus) | Weak overlap, different scope |
| `LanguageModel` | An LLM used by an AI agent | Implicit in `actorId` model-family | No conflict |
| `Agent` | "Entity capable of perceiving, processing, acting to achieve goals" | Worker (engine) / `actorId` (ledger) | Different levels, coexist cleanly |

---

## Key Findings

### casehub's vocabulary is well-grounded

Every major casehub concept has a clear and justified position in the AgentO taxonomy. The mappings are clean; there are no surprising gaps or contradictions.

### The `Task` level distinction is correct

casehub's three-level hierarchy (Work → WorkItem / Task) is more precise than AgentO, which uses `Task` at the Work level. casehub's naming decision (ADR-0003) is vindicated: keeping `Task` at sub-step level avoids exactly the collision AgentO creates by using it at the Work level.

### `Goal` — casehub is more precise

AgentO defines Goal as "a desired state." casehub's `Goal` is a formal evaluable completion expression. This is the stronger definition, and it converges independently with langchain4j's `GoalOrientedPlanner` (noted in engine ADR-0007), which also treats goals as evaluable dependency conditions over output keys.

### `Milestone` — casehub contribution to AgentO

casehub has `Milestone` (intermediate checkpoint, `MilestoneReached` event) with no equivalent in AgentO. Clean contribution.

---

## Where casehub is Ahead — The Normative Layer

AgentO's normative model: `Constraint` as a subclass of `KnowledgeBase` — stored rules that restrict or guide agent behaviour. This is a knowledge-representation concept, not a deontic model.

casehub-qhorus has a full 4-layer normative accountability framework grounded in Austin/Searle speech act theory:

1. **Illocutionary** — what was said: 9 speech-act types (Directive, Commissive, Assertive, Declarative, Expressive, etc.)
2. **Commitment** — what was obligated: `Commitment` entity with 7-state lifecycle (OPEN → FULFILLED / FAILED / EXPIRED / ...)
3. **Temporal** — when obligations become stale: Watchdog, deadline enforcement
4. **Enforcement** — casehub-engine orchestration reacts to commitment outcomes via CDI events

The 3-channel normative layout (`work` / `observe` / `oversight`) structures agent-to-agent and agent-to-human interactions. This is a genuine deontic model — obligations, commitments, temporal validity — that AgentO does not have at all.

**Contribution direction:** casehub's normative model is what we'd contribute to AgentO, not import from it.

---

## Contribution Opportunities

| What casehub has | What AgentO lacks | Contribution |
|---|---|---|
| `Goal` as evaluable expression | `Goal` as desired-state string only | Stronger Goal definition with evaluation mechanism |
| `Milestone` | Nothing | New class: intermediate checkpoint with event semantics |
| 4-layer normative model | `Constraint` as KnowledgeBase only | Full deontic layer: speech act types, Commitment lifecycle, temporal enforcement |
| `Case` as formal ACM instance | `Objective` as informal team goal | Formal Case/CaseDefinition with completion semantics |
