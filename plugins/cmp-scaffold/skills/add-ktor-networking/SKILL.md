---
name: add-ktor-networking
description: Replace a mock remote data source with a real Ktor HTTP client implementation. Use after add-remote-datasource has created the remote interface and mock.
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Ktor Networking

Replace `Mock<Feature>RemoteDataSource` with a real Ktor HTTP client implementation. See [networking-patterns.md](../../references/networking-patterns.md) for templates.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `products`, `catalog`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Validate Existing Module

Verify the feature has `Mock<Feature>RemoteDataSource` in `<feature>/data/impl/datasource/remote/`. If missing, tell the user to run `/cmp-scaffold:add-remote-datasource <feature>` first.

### 3. Read Existing Files

Read the remote data source interface, model classes, and DI module to understand the API surface.

### 4. Set Up Core Network Module (if needed)

Check if `core/network/` exists (glob for `**/core/network/**/*.kt`). If missing:
- Create `core/network/` module with `createHttpClient()`, `safeApiCall()`, and `networkModule` using templates from [networking-patterns.md](../../references/networking-patterns.md)
- Add Ktor version catalog entries to `gradle/libs.versions.toml`
- Register `core:network` in `settings.gradle.kts`
- Add `networkModule` to AppModule `includes(...)`

### 5. Create DTO Classes

Create `<feature>/data/impl/datasource/remote/dto/<Model>Dto.kt`:
- `@Serializable` data class mirroring the API response shape
- `toDomain()` extension mapping DTO → domain model

### 6. Create Ktor Remote Data Source

Create `Ktor<Feature>RemoteDataSource` implementing the existing `<Feature>RemoteDataSource` interface. Use `safeApiCall { }` for all network operations. Use `httpClient.get()` / `.body<T>()` pattern.

### 7. Update DI Module

Swap the binding from `Mock*` to `Ktor*`:
```kotlin
singleOf(::Ktor<Feature>RemoteDataSource) bind <Feature>RemoteDataSource::class
```

### 8. Update build.gradle.kts

Add Ktor dependencies to `<feature>/data/impl/build.gradle.kts`:
- commonMain: `ktor-client-core`, `ktor-client-content-negotiation`, `ktor-serialization-kotlinx-json`, `ktor-client-logging`
- androidMain: `ktor-client-okhttp`
- iosMain: `ktor-client-darwin`

Add `project(":core:network")` dependency if core network module was created.

### 9. Preserve Mock

Keep `Mock<Feature>RemoteDataSource` in the same file — useful for testing and as API documentation.

### 10. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `Ktor<Feature>RemoteDataSource` implements the existing remote interface
- [ ] All network calls use `safeApiCall { }` wrapper
- [ ] DTO classes are `@Serializable` with `toDomain()` extension
- [ ] DI binds `Ktor*` (not `Mock*`) to remote data source interface
- [ ] Ktor dependencies added via version catalog references (not hardcoded versions)
- [ ] `core:network` registered in `settings.gradle.kts` if newly created

### 11. Report

Output a summary listing files created, files modified, and next steps:
- Configure base URL (environment-based or build config)
- Add authentication headers if needed
- `/cmp-scaffold:add-room-database <feature>` for persistent local storage (replaces InMemory with Room)
- `/cmp-scaffold:scaffold-tests <feature>` to generate tests for the networking layer
- `/cmp-scaffold:polish-ui <feature>` to add pull-to-refresh and rich visual design
