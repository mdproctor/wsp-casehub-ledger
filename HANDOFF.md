# Session Handover — 2026-06-15

## Last Session

Closed two issues: #100 (concurrent write safety — `LedgerSequenceAllocator` INSERT ON CONFLICT + per-subject lock in InMemory, concurrent PgIT) and #138 (@DefaultBean no-op repositories — `NoOpLedgerEntryRepository`, `NoOpActorIdentityBindingRepository`, `JpaActorIdentityBindingRepository @Alternative`). Both squashed, pushed to fork and casehubio/ledger main. Four multi-round spec reviews preceded each implementation.

## Immediate Next Step

`/work` — pick next open issue. Run `gh issue list --repo casehubio/ledger --state open` to see current list (9 open).

## What's Left

- **Merkle Serialization Invariant protocol** — document the three-fact invariant in `JpaLedgerEntryRepository.save()` as a casehub-ledger garden protocol. Covered in ARC42STORIES §10 and Javadoc; formal protocol entry not yet created. · S · Low
- **GE-20260605-b734b3 REVISE** — add `@ConfigProperty(name="quarkus.datasource.db-kind")` dialect detection as alternative to pure MERGE. · XS · Low
- **Consumer exclude-types cleanup** — `casehub-work` and `casehub-engine` can drop `quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.identity.**` after #138 ships. Cross-repo; not our branch. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #96 | Code-gen reactive service tier (Vert.x codegen style) | L | High | Parked — wait until pair count warrants it |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | Unblocked (#85 closed) |
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | Unblocked (#85 closed) |
| #123 | Engine-side TrustScoreSource migration | M | Low | Cross-repo (casehub-engine); unblocked (#118 closed) |
| #126 | Decouple MCP telemetry from MessageType.EVENT content | — | — | qhorus concern |
| #136 | TrustGateService batch scoring | — | — | CBR; needs engine#476 |
| #139 | Merkle frontier not tenant-scoped | M | Med | Filed this session; nameUUID subjectId collision |
| #141 | Named datasource dialect detection in LedgerSequenceAllocator | S | Low | Filed this session |
| #143 | JpaActorTrustScoreRepository @Alternative | S | Low | Filed this session; same pattern as #138 |

## References

- Specs: `specs/issue-100-concurrent-write-safety/`, `specs/issue-138-noop-defaultbean-ledger-repo/`
- ARC42STORIES.MD updated this session (§5, §9.4·Audit Primitives, §10, §12)
- Garden: GE-20260615-6d0ae3 submitted (Merkle Serialization Invariant — undocumented)
