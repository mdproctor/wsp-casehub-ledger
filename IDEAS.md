# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## ~~2026-04-23 — Submission target: Quarkiverse vs SmallRye~~

**Status:** resolved — not applicable

`casehub-ledger` is a CaseHub sub-project, permanently homed in the `casehubio` GitHub org.
External submission (Quarkiverse, SmallRye) is not planned.

---

## 2026-04-16 — `/auditability-check` Claude Code skill

**Priority:** medium
**Status:** active

A Claude Code slash command that applies the 8-axiom auditability framework
(ACM FAIR 2025 — Integrity, Coverage, Temporal Coherence, Verifiability,
Accessibility, Resource Proportionality, Privacy Compatibility, Governance
Alignment) to any codebase and outputs a structured `AUDITABILITY.md` gap
analysis with a priority map. Could be parameterised by compliance lens:
`8-axiom-auditability`, `article-12`, `gdpr-art-22`, `nist-ai-rmf`.

**Context:** Emerged from the casehub-ledger research session (2026-04-16)
after producing `docs/AUDITABILITY.md`. The pattern — load a framework,
assess a codebase, apply a design constraint, output a structured gap
analysis — is general enough to offer as a cc-praxis skill alongside
`java-security-audit` and `python-security-audit`. Timely given EU AI Act
enforcement deadline of 2 August 2026.

**Promoted to:**

---

## 2026-05-15 — Bilateral entry signing (agent non-repudiation)

**Priority:** medium-high
**Status:** active

Each `LedgerEntry` is currently signed by the ledger (via Merkle/Ed25519 checkpoints), proving the entry hasn't been tampered with. But the *agent* that produced the entry never signs it — so an agent could later deny the action. Research term: "Verifiable Interaction Ledger" (bilateral signing by both parties).

Add an optional `agentSignature` field to `LedgerEntry`: the agent signs the canonical leaf hash (already defined: `subjectId|seqNum|entryType|actorId|actorRole|occurredAt`) with its own private key before submission. The ledger stores the signature and the agent's public key reference. Verification is independent of the Merkle chain.

Dovetails with: existing canonical form, existing Ed25519 infrastructure (`LedgerMerklePublisher`), existing `actorId` model. Low schema change (one nullable column). High compliance value — non-repudiation is an explicit Article 12 requirement.

**Promoted to:**

---

## 2026-05-15 — Agent DID/VC cryptographic identity binding

**Priority:** medium
**Status:** active

Ledger's `actorId` format (`claude:reviewer@v1`) is a convention, not a cryptographic binding. Any agent can claim any `actorId`. The field is believed, not verified. The field is moving toward DIDs (Decentralized Identifiers) and VCs (Verifiable Credentials) — an agent proves its identity by presenting a signed credential rather than asserting a string.

A lightweight first step: an optional `ActorIdentityProvider` SPI implementation that validates `actorId` against a VC at registration time (rather than blocking every write). Full DID resolution is a larger investment.

Dovetails with: existing `ActorIdentityProvider` SPI, existing `ActorIdentity` pseudonymisation model, `agentSignature` idea above (signatures require a key, keys require identity).

**Promoted to:**

---

## 2026-05-15 — Zero-knowledge compliance proofs (Proof of Conduct)

**Priority:** low-medium (research horizon)
**Status:** active

Merkle inclusion proofs prove an entry *exists* in the ledger. They don't prove the agent's *behaviour was lawful* under a policy. The Aegis architecture (2025) calls this a "Proof of Conduct" — a zk-STARK-based cryptographic statement that behaviour was lawful under a policy layer, without exposing the content.

Directly relevant to EU AI Act Article 12: regulators want verifiable compliance without access to proprietary model internals or sensitive case data. Frameworks: zk-MCP (prove agent communication audit without exposing content), ZKMLOps (prove ML compliance without exposing model). High complexity — GPU-accelerated proving required for large models. Natural extension of the existing Merkle + Ed25519 foundation.

**Promoted to:**

---

## 2026-05-15 — AntTrust as alternative to EigenTrust

**Priority:** low
**Status:** active

Ledger uses EigenTrust for transitive global trust scores (`EigenTrustComputer`). Research benchmark (2025): AntTrust outperforms EigenTrust, TNA-SL, and TACS on success rate stability and malicious peer resistance. EigenTrust degrades in dynamic networks and is exploitable by coordinated malicious peers gaming their feedback.

For internal CaseHub deployments with known, trusted agents this is not urgent. For open or adversarial networks (external agent meshes, federated deployments) the limitation matters. Action: ADR noting the EigenTrust limitation and conditions under which AntTrust migration is warranted. No code change yet.

**Promoted to:**

---

## 2026-05-15 — Delegation chain tracking (RFC 8693 `act` claim)

**Priority:** medium
**Status:** active

When agent A spawns agent B, ledger captures the causal link via `causedByEntryId`, but not the *delegation* — who authorised whom to act, with what scope. RFC 8693 defines the `act` claim for exactly this: the acting party, the delegating party, and the scope of the mandate. As multi-agent orchestration grows (casehub-assisteddev, claudony agent meshes), reconstructing the full delegation chain becomes a compliance requirement.

A `DelegationSupplement` (new `LedgerSupplement` subclass) with `delegatedByActorId`, `delegatedScope`, and `mandateReference` would close the gap without touching the core model. Dovetails with: existing `LedgerSupplement` JOINED inheritance, existing `causedByEntryId`, existing `ProvenanceSupplement` pattern.

**Promoted to:**
