---
name: add-remote-datasource
description: Add a mock remote data source to an existing feature module, upgrading it from local-only to local+remote with cache-first pattern. Use add-ktor-networking afterward to replace the mock with a real HTTP client.
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Remote Data Source

Upgrade an existing feature module from local-only to the full local+remote cache-first pattern. See [code-templates.md](../../references/code-templates.md) for the remote data source and cache-first repository templates.

## Input

`$ARGUMENTS` — the feature name (e.g., `cart`, `favorites`, `settings`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`).

### 2. Validate Existing Module

Verify local-only pattern exists: `<feature>/data/api/` with repository interface, `<feature>/data/impl/datasource/local/` with LocalDataSource + InMemory impl, `<feature>/data/impl/repository/` with RepositoryImpl, `<feature>/data/impl/di/` with data module. If missing, tell user to run `/cmp-scaffold:scaffold-feature <feature>`.

### 3. Read Existing Module Files

Read all data layer files to understand model classes, repository interface, local data source interface, and current repository implementation.

### 4. Create Remote Data Source

Create `<feature>/data/impl/.../datasource/remote/<Feature>RemoteDataSource.kt` using the template from [code-templates.md](../../references/code-templates.md#remote-data-source-interface--mock-impl-in-same-file). Key rules: interface + mock in same file, all ops return `Result<T>`, mock includes `simulateNetworkDelay()`.

### 5. Add refresh to Repository Interface

Add `suspend fun refresh<Feature>s(): Result<Unit>` if not present.

### 6. Update Repository Implementation

Update using the cache-first template from [code-templates.md](../../references/code-templates.md#repository-implementation-local--remote-cache-first). Preserve existing methods. Add remote data source to constructor + refresh method.

### 7. Update DI Module

Add `singleOf(::Mock<Feature>RemoteDataSource) bind <Feature>RemoteDataSource::class`.

### 8. Update ViewModel (optional)

Add `init { refresh() }` if the module benefits from auto-refresh on screen load.

### 9. Ensure Local Insert Methods

Add `suspend fun insert<Feature>s(items: List<Model>)` to interface and implementation if missing.

### 10. Verify

Before reporting, confirm each item — fix any violations:

- [ ] Remote data source interface + mock in same file
- [ ] All remote ops return `Result<T>`
- [ ] Mock includes `simulateNetworkDelay()`
- [ ] Repository constructor takes both local and remote data sources
- [ ] `refresh<Feature>s()` added to repository interface and implementation
- [ ] DI module binds `Mock<Feature>RemoteDataSource` to interface

### 11. Report

Output a summary listing files created, files modified, and the coordination strategy (cache-first). Suggest next steps:
- `/cmp-scaffold:add-ktor-networking <feature>` to replace the mock with a real Ktor HTTP client
- `/cmp-scaffold:scaffold-tests <feature>` to generate tests for the cache-first data layer
- `/cmp-scaffold:polish-ui <feature>` to add pull-to-refresh and rich visual design
