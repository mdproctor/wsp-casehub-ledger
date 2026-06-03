# Design Journal — issue-99-native-image-flyway-resources

### 2026-06-03 · §10

`LedgerProcessor` now produces `NativeImageResourcePatternsBuildItem` registering
`db/ledger/migration/*.sql` for inclusion in native image builds. Without this
`@BuildStep`, GraalVM's resource collection phase silently omits the SQL files —
classpath scanning that works at JVM runtime is replaced at native-image time by
a pre-registered list, and any resource not in that list is absent in the binary.
The deployment module is the correct place for this registration: it is the
build-time processor that knows which classpath paths the extension owns, and
`NativeImageResourcePatternsBuildItem` is a deployment-module build item.
Consumers do not need to add anything — the extension self-registers its own resources.
