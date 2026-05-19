# Design Journal — issue-92-optional-reactive-repo

### 2026-05-19 · §Architecture

Blocking and reactive service tiers are now separate `@ApplicationScoped` beans.
`LedgerVerificationService` and `KeyRotationService` carry zero reactive imports —
they build and run correctly in JDBC-only consumers. `ReactiveLedgerVerificationService`
and `ReactiveKeyRotationService` hold all `Uni<T>`-returning methods. The reactive tier
is excluded from CDI at Quarkus augmentation by `LedgerProcessor.excludeReactiveBeans`
(`ExcludedTypeBuildItem`) when `casehub.ledger.reactive.enabled=false` (default).
`BlockingTierPurityTest` enforces the separation at build time via reflection — checking
both method return types and field injection types on blocking-tier beans.

### 2026-05-19 · §Key Design Decisions

Three alternatives to the `ExcludedTypeBuildItem` approach were evaluated and rejected.
`Instance<T>` optional injection moves the failure from build time to runtime and leaves
mixed-tier service beans intact — the code smell remains. `@DefaultBean` no-op defaults
in production would conflict with the existing `@DefaultBean` blocking test shims, causing
CDI ambiguity in `@QuarkusTest`. `@IfBuildProperty` on runtime beans requires the property
to be declared as `@ConfigRoot(BUILD_TIME)` in the deployment module to be reliable; it
was replaced by `ExcludedTypeBuildItem` in `LedgerProcessor`, which is the canonical
Quarkus extension pattern for conditional bean registration. `LedgerConfig.ReactiveConfig`
is retained in the runtime config root solely to satisfy Quarkus config validation
(`SRCFG00050`) — it is not the authoritative gate; `LedgerBuildTimeConfig` in the
deployment module is.
