---
layout: post
title: "Closing the Snapshot Gaps"
date: 2026-08-28
entry_type: note
subtype: diary
projects: [casehubio/ledger]
tags: [trust-scores, snapshots, compliance]
---

# Closing the Snapshot Gaps

The trust score snapshot table landed back in #183 as a basic foundation — GLOBAL
and CAPABILITY scores captured on each computation run, with two query methods.
Enough to prove the concept, not enough for qhorus's compliance evidence export
to build a Trust Score History Report on top of.

Issue #203 listed the gaps: no `scoreType` discriminator (queries inferred type
from null checks on `capabilityTag`), no `dimensionKey` column (DIMENSION and
CAPABILITY_DIMENSION scores weren't captured at all), no time-range query, and
no retention policy. The compliance report needs all of these — without them,
it falls back to current-score-only mode with no trajectory data.

The interesting architectural question was tenancyId. The issue spec listed it as
a column, but trust computation is cross-tenant by design — `TrustScoreJob` pulls
from `CrossTenantLedgerEntryRepository`, and `ActorTrustScore` itself has no
tenancy scope. Adding tenancyId to snapshots without adding it to the score entity
would be inconsistent — you'd be tagging a cross-tenant computation result with a
tenant ID that has no meaning. The protocol that requires tenancyId on per-subject
tables (PP-20260616) doesn't apply here either; snapshots are keyed by actor, not
subject. If per-tenant trust ever becomes a thing, the right sequence is tenancyId
on `ActorTrustScore` first, snapshots follow.

The implementation was clean. `scoreType` and `dimensionKey` go into the V1000
migration (no production database — schema convention says rewrite in place).
`PerActorTrustComputer` already had the right structure — the DIMENSION and
CAPABILITY_DIMENSION loops just needed snapshot saves wired in, with previous-score
lookups mirroring what the CAPABILITY loop already did. A `deleteOlderThan` method
on the repository SPI gives retention, wired into `TrustScoreJob` with a default
365-day window. Set to 0 to keep everything.

The JPQL for the new queries uses fully-qualified enum literals in `@NamedQuery`
annotations rather than parameter binding — `s.scoreType = io.casehub.ledger.api.model.ScoreType.GLOBAL`
instead of a `:scoreType` parameter. Slightly unusual, but it means the query
is self-describing and doesn't need an extra `setParameter` call. Hibernate
validates it at boot time either way.

The snapshot table is now a genuine time-series store for trust trajectories —
all four score types captured, queryable by time range, with automatic retention
trimming. Qhorus can build the full compliance report without falling back to
the degraded single-point mode.
