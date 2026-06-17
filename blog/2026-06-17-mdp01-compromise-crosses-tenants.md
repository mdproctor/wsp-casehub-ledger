---
layout: post
title: "The Compromise That Was Already Global"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [multi-tenancy, key-rotation, security, repository-design]
---

`KeyRotationRepository` had the same structural gap as the identity binding repository from the previous session: `findByActorId` returned results from any tenant. The fix looked identical too — add `tenancyId`, update the named query, propagate through the service layer and implementations.

But `KeyRotationRepository` has two read methods, not one. And they don't work the same way.

`findByActorId` is a history query: show me the key rotation events for this actor in my tenancy. That one clearly needs `tenancyId`. No tenant should see another's rotation records.

`findCompromisedByActorIdAndKeyRef` is different. It answers a security question: has this signing key ever been reported as compromised? That question has no tenant scope. A private key is either compromised or it isn't. Two tenants sharing the same LLM persona — `claude:reviewer@v1`, say — share the key pair too. If one tenant reports the key as COMPROMISED, every signature from that key pair should be treated as SUSPECT, regardless of which tenant is asking. Scoping compromise detection per-tenant would mean a security incident in one deployment has no effect on another, even when both are trusting the same private key.

The issue I filed had three options. Add `tenancyId` to both methods; tenant-scope history but leave compromise detection cross-tenant; or split into two repository interfaces. The right answer was the middle one, but what confirmed it was the code that already existed.

`AgentSignatureVerificationService.compromisedEffectiveSince()` calls `keyRotationService.compromisedWindows(actorId, keyRef)` — no `tenancyId` in sight. That method was written when `KeyRotationRepository` was first designed, before the multi-tenancy work landed. The intent was already there: compromise detection was always global. The fix for this issue wasn't a new design choice; it was formalising what the code had assumed from the start.

We updated `findByActorId` to take `tenancyId`, rewrote the named query to filter on it, and pushed the parameter through the service layer, both reactive variants, the JPA implementation, the in-memory implementation, and the test shim. The change to `findCompromisedByActorIdAndKeyRef`: none.

The cross-tenant isolation tests are the most useful part. One seeds the same actor in two tenants and confirms `findByActorId` returns only the caller's records. Another seeds a COMPROMISED rotation in tenant A and confirms it's visible globally. Whether you query from tenant B, or from no tenant context at all, the key is compromised. The query finds it.

---

CI was red when the session started. The `consumer-compat-test` module was failing on deploy — standalone POM, no parent, no `<distributionManagement>`, nowhere to publish. The one-line fix was already on the fork. CI kept running against the old SHA because the commit hadn't been pushed to the upstream repository. `git ls-remote origin main` showed the fix was there; `git ls-remote upstream main` showed it wasn't. The fork is what you see; the blessed repo is where CI runs.
