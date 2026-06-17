# Design Journal — issue-146-key-rotation-tenancy

### 2026-06-17 · §10 Architectural Decisions

Key rotation history (`findByActorId`) is tenant-scoped: each tenant sees
only the rotation records it recorded for a given actor. This isolates
operational audit trails across tenants even when they share an actorId
(e.g. a shared LLM persona like `claude:reviewer@v1`).

Compromise detection (`findCompromisedByActorIdAndKeyRef`) remains cross-tenant
by deliberate design. A compromised signing key is a global security fact —
if any tenant reports a key as COMPROMISED, all tenants must treat signatures
from that key as SUSPECT. This matches the existing behaviour in
`AgentSignatureVerificationService.compromisedEffectiveSince()`, which already
passes no tenancyId into the compromise check. Cross-tenant reads on this
method are a security feature, not a gap.
