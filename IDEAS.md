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
