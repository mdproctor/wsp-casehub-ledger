# Design Journal — issue-104-in-memory-agent-signer

### 2026-06-03 · §10

`InMemoryAgentSigner` added to `persistence-memory` as `@Alternative @Priority(1) @ApplicationScoped`.
Follows the module's displacement pattern: `ConfiguredAgentSigner @DefaultBean` is the
production bean; `InMemoryAgentSigner` displaces it in tests via `quarkus.arc.selected-alternatives`.
The implementation stays trivial (no caching, no lifecycle hooks) because its sole purpose
is test isolation — tests call `register()` and `clear()` explicitly, which is more readable
than auto-wiring or lifecycle-scoped setup. Signing delegates to `AgentSignature.signWith()`,
preserving algorithm transparency (PP-20260523-e7b577).
