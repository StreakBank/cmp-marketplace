---
name: add-background-sync
description: Add background sync capability to an existing feature module using WorkManager (Android) and BGTaskScheduler (iOS)
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Background Sync

Add background sync to a feature module so data refreshes automatically when the app is backgrounded or on a schedule. See [background-work-patterns.md](../../references/background-work-patterns.md) for templates.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `catalog`, `feed`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Validate Prerequisites

Verify the feature has a cache-first repository with `refresh<Feature>s()` in `<feature>/data/api/repository/`. If missing, tell the user to run `/cmp-scaffold:add-remote-datasource <feature>` first — background sync requires a remote data source to sync from.

### 3. Set Up Core Sync Module (if needed)

Check if `core/sync/` exists (glob for `**/core/sync/**/*.kt`). If missing, create the full `core/sync/` module using templates from [background-work-patterns.md](../../references/background-work-patterns.md):
- `SyncScheduler` interface (commonMain)
- `AndroidSyncScheduler` (androidMain) — WorkManager implementation
- `IosSyncScheduler` (iosMain) — BGTaskScheduler implementation
- `SyncWorker` (androidMain) — CoroutineWorker routing to SyncRegistry
- `SyncUseCase` interface (commonMain)
- `SyncRegistry` (commonMain)
- `retryWithBackoff` utility (commonMain)
- `syncModule` and expect/actual `platformSyncModule` Koin modules
- `core/sync/build.gradle.kts` with WorkManager dependency for androidMain
- Add `work-runtime-ktx` version catalog entry if not present
- Register `core:sync` in `settings.gradle.kts`
- Add `syncModule` and `platformSyncModule` to AppModule `includes(...)`

### 4. Create Sync Use Case

Create `<feature>/domain/usecases/Sync<Feature>UseCase.kt` implementing `SyncUseCase`:
- Constructor-inject the `<Feature>Repository`
- In `sync()`, call `retryWithBackoff { repository.refresh<Feature>s().getOrThrow() }`
- If `<feature>/domain/` does not exist, tell the user to run `/cmp-scaffold:add-domain-layer <feature>` first, or create a minimal domain module following [domain-layer-patterns.md](../../references/domain-layer-patterns.md)

### 5. Register Sync Use Case

Update the feature's domain DI module to:
- Register `Sync<Feature>UseCase` with `factoryOf`
- Call `get<SyncRegistry>().register("<feature>_sync", get<Sync<Feature>UseCase>())` in a `single { }` block

### 6. Add Dependency on Core Sync

Add `project(":core:sync")` to the feature's domain module `build.gradle.kts` dependencies.

Add `project(":core:sync")` to `composeApp/build.gradle.kts` if not present.

### 7. Document iOS Setup

Output the required iOS configuration as next steps (cannot be automated from Kotlin):
- `BGTaskScheduler.shared.register(forTaskWithIdentifier:)` in AppDelegate
- `BGTaskSchedulerPermittedIdentifiers` array in `Info.plist`
- Provide the exact code and plist entries from [background-work-patterns.md](../../references/background-work-patterns.md)

### 8. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `SyncScheduler` interface exists in `core/sync/` with `scheduleOneTime`, `schedulePeriodic`, `cancel`
- [ ] `AndroidSyncScheduler` uses `Constraints` with `NetworkType.CONNECTED`
- [ ] `SyncWorker` extends `CoroutineWorker` and routes via `SyncRegistry`
- [ ] `Sync<Feature>UseCase` implements `SyncUseCase` and uses `retryWithBackoff`
- [ ] Feature domain DI module registers sync use case with `SyncRegistry`
- [ ] `core/sync/build.gradle.kts` includes `work-runtime-ktx` in androidMain
- [ ] Version catalog has `work-runtime-ktx` entry
- [ ] `syncModule` and `platformSyncModule` included in AppModule

### 9. Report

Output a summary listing files created, files modified, and next steps:
- **iOS setup required** — BGTaskScheduler registration in AppDelegate + Info.plist identifiers (provide exact code)
- Trigger sync from ViewModel: `syncScheduler.scheduleOneTime("<feature>_sync")`
- Schedule periodic sync on app launch: `syncScheduler.schedulePeriodic("<feature>_sync", intervalMinutes = 60)`
- `/cmp-scaffold:scaffold-tests <feature>` to generate tests (mock `SyncScheduler` interface)
- `/cmp-scaffold:add-room-database <feature>` if local persistence is not yet set up

## Troubleshooting

- **WorkManager not found** — verify `work-runtime-ktx` is in version catalog and `core/sync/build.gradle.kts` androidMain dependencies
- **BGTaskScheduler tasks not firing** — iOS controls timing; ensure `BGTaskSchedulerPermittedIdentifiers` in Info.plist matches the tag string exactly
- **SyncWorker fails immediately** — check that `SyncRegistry.register()` is called before the worker runs; Koin module ordering matters
- **Retry loops on permanent failures** — `retryWithBackoff` retries transient errors; for 4xx client errors, catch and return early in the use case
