# CaseHub Ledger — Session Handover
**Date:** 2026-06-11

## Current State

On branch `issue-128-merkle-content-integrity`. Implementation complete (10 commits, 30 files, 984 insertions). Pre-close sweep done (forage: 4 garden entries, diary written). **Not yet squashed, rebased, or pushed.** Work-end remaining steps: squash, rebase onto main, push to fork, deliver to blessed repo, close #128.

## What Landed This Session

- **#128**: Merkle leaf hash expanded from 6 structural fields to full content coverage — supplementJson, tenancyId, actorType, causedByEntryId, plus subclass domain content via `domainContentBytes()`. `canonicalBytes()` moved from `LedgerMerkleTree` (static) to `LedgerEntry` (public final instance method). Save pipeline restructured: prepareKey → enrich → hash → sign → persist. `AgentSignatureEnricher` deleted — replaced by `AgentEntrySigner` (direct call, not enricher). Missing `@Priority` annotations fixed on TraceIdEnricher/ProvenanceCaptureEnricher. Build-time guard enforces `domainContentBytes()` override on `@Entity` subclasses with persistent fields.
- **#135**: CI fix — `ScimActorDIDProviderTest` HTTPS validation using 5-arg constructor + `@CrossTenant` qualifier on InMemory injection sites.
- **#133, #134**: Closed — no ledger-side work (engine#460 is the fix).
- **#108, #110**: Transferred to `casehubio/platform` (platform#84, platform#85) — identity SPIs live there now.
- **#127 spec**: Updated to reflect #131/#132 CDI changes (status → Implemented, §3 rewritten).
- Cross-repo coherence review completed for multi-tenancy specs (ledger/work/qhorus).

## Immediate Next Step

Run `work-end` to complete #128 close: squash → rebase → push → close issue.

## What's Left

- Backup branches accumulating — oldest eligible past 14-day hold: `20260507`, `20260508`, `20260521`, `20260522` · XS · Low
- Cross-repo coherence review: ledger#127 spec + work + qhorus tenancy specs · M · Med (spec updates applied, implementation items remain in other repos)
- engine#460: remove tenancyId field from CaseLedgerEntry/WorkerDecisionEntry · S · Low
- engine#471: add `domainContentBytes()` to CaseLedgerEntry · XS · Low (blocked by #128 landing)
- qhorus#270: add `domainContentBytes()` to MessageLedgerEntry · XS · Low (blocked by #128 landing)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #130 | refactor: exempt ActorType.SYSTEM from tokenisation | S | Low | Unblocked |
| #126 | Decouple MCP tool telemetry from EVENT content | M | Med | Possibly misfiled (mostly qhorus) |

## References

| Artifact | Where |
|----------|-------|
| Spec | `specs/issue-128-merkle-content-integrity/2026-06-11-merkle-content-integrity-design.md` |
| Plan | `plans/2026-06-11-merkle-content-integrity.md` |
| Blog entry | `blog/2026-06-11-mdp01-the-ledger-that-proved-nothing.md` |
