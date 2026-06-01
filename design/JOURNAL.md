# Design Journal — issue-94-parity-enforcement + issue-112-113-identity-extraction

### 2026-05-22 · §Architecture · §Key Design Decisions · §Implementation Tracker

The blocking/reactive tier separation (protocol PP-20260519-39a9a5) was previously
enforced by convention only: the protocol document stated the parity rule, but nothing
in the build verified it. This commit makes enforcement structural via an ArchUnit
bytecode scan in `BlockingReactiveParityTest`. The test auto-discovers all
`Reactive*Service` classes by naming convention — no annotation required on production
code — and enforces bidirectional parity: every public non-static method `foo()` in the
blocking tier must have `fooAsync()` in the reactive tier (and vice versa), with matching
raw parameter types. A minimum-count guard prevents vacuous green on package rename.
ArchUnit was chosen over plain reflection (no generalisation, no overload safety) and
`@BuildStep` (fires at consumer deploy time, not extension test time).

**§Implementation Tracker row to add** (after "Reactive service tier separation"):
`| **Blocking/reactive parity enforcement** | ✅ Done | `BlockingReactiveParityTest` (ArchUnit 1.4.1 bytecode scan); auto-discovers all `Reactive*Service` classes by naming convention; bidirectional method parity with parameter-type checking; minimum-count guard against vacuous pass on package rename; `Uni<T>` return type enforcement. 449 tests. Closes #94. |`

---

### 2026-06-01 · §Ecosystem Context · §Key Design Decisions · §Implementation Tracker

**Identity infrastructure extracted to casehub-platform-identity (Phase 1 + Phase 2).**

The identity SPI pipeline (ActorDIDProvider, DIDResolver, AgentCredentialValidator), all
model types (DIDDocument, VerificationMethod, IdentityVerificationResult,
CredentialValidationResult, IdentityBindingStatus), CDI events
(AgentIdentityValidatedEvent, AgentIdentityViolationEvent), and all implementations
(NoOp*, KeyDIDResolver, WebDIDResolver, ConfiguredActorDIDProvider, ScimActorDIDProvider,
AbstractCachingIdentityProvider) moved from casehub-ledger to casehub-platform-identity.

**Rationale:** Ledger accumulated identity infrastructure by development order, not
design ownership. Ledger's domain is audit; it uses identity to verify who wrote entries
but does not own identity. Any module needing agent identity resolution should not depend
on the full ledger stack. This motivated platform ownership rule PP-20260531-dd7062.

**§Ecosystem Context update:** casehub-platform-identity now sits below casehub-ledger
in the dependency graph. Ledger depends on it as a consumer.

**§Key Design Decisions:**

1. **Platform ownership vs development order** — a module owns infrastructure only if it
   is the definitive provider of that capability. Ledger was first to implement identity
   because it was the first consumer, not because it owns it. Protocol PP-20260531-dd7062
   formalises the rule for all future placement decisions.

2. **IdentityCacheInvalidator bridge pattern** — ScimActorDIDProvider (platform) no longer
   observes AgentKeyRotatedEvent (ledger) to avoid a backwards dependency from platform to
   ledger. Ledger adds a thin IdentityCacheInvalidator bean that observes the ledger event
   and calls actorDIDProvider.invalidate(actorId) if the active provider is an
   AbstractCachingIdentityProvider. Pattern: event stays in owning domain; consuming domain
   provides the bridge.

3. **Primitive signature for verification services (Phase 2)** — AgentIdentityVerificationService
   was decoupled from LedgerEntry by extracting the three fields it reads (actorId, actorDid,
   agentPublicKey). Ledger retains thin adapter classes preserving the existing consumer API
   (same class names, same LedgerEntry parameter) while the platform service is domain-agnostic.

**§Implementation Tracker rows to add:**

`| **Identity infrastructure — platform extraction (Phase 1)** | ✅ Done | casehub-platform-identity: SPIs, model, CDI events, all implementations; config prefix casehub.identity.*; IdentityCacheInvalidator bridge; 546 tests. Closes ledger#112, platform#52. |`

`| **Identity verification services — platform extraction (Phase 2)** | ✅ Done | AgentIdentityVerificationService + ReactiveAgentIdentityVerificationService in platform with primitive signatures; ledger thin adapters retain existing consumer API. Closes ledger#113, platform#53. |`
