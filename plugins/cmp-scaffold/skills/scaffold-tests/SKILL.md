---
name: scaffold-tests
description: Generate unit tests for a feature module. Use after adding business logic to ensure ViewModel, Repository, and UseCase layers are testable.
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Scaffold Tests

Generate unit test files for an existing feature module.

## Input

`$ARGUMENTS` — the feature name (e.g., `products`, `cart`, `settings`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`).

### 2. Analyze the Module

Read all source files to understand: ViewModels and their dependencies, UiState types, Repository methods (Flow reads, suspend writes), data sources, and whether a domain layer exists.

Files to read:
- `<feature>/feature/.../feature/*ViewModel.kt`, `*UiState.kt`
- `<feature>/data/api/.../repository/*Repository.kt`, `.../model/*.kt`
- `<feature>/data/impl/.../datasource/**/*.kt`
- `<feature>/domain/.../usecases/*.kt` (if exists)

### 3. Add Test Dependencies

Add to each module's `sourceSets` block if missing — see [test-patterns.md — Test Dependencies](../../references/test-patterns.md#test-dependencies-buildgradlekts).

### 4. Create Test Doubles

Use templates from [test-patterns.md](../../references/test-patterns.md):
- **Fake Local Data Source** — mirrors interface with MutableStateFlow + `emit()`/`clear()` helpers
- **Fake Remote Data Source** (if exists) — with `shouldFail` control
- **Fake Repository** — mirrors interface with `shouldFail`/`failureMessage` control

### 5. Create ViewModel Tests

Use [test-patterns.md — ViewModel Test](../../references/test-patterns.md#viewmodel-test). Generate tests for: initial Loading state, Success with data, Empty (if applicable), each action method (success + failure), errorEvents emission on failure.

### 6. Create Repository Tests

Use [test-patterns.md — Repository Test](../../references/test-patterns.md#repository-test). If remote exists, also test cache-first behavior, refresh success, refresh failure.

### 7. Create UseCase Tests (if domain exists)

Use [test-patterns.md — UseCase Test](../../references/test-patterns.md#usecase-test-if-domain-layer-exists).

### 8. Verify

Before reporting, confirm each item — fix any violations:

- [ ] Test dependencies added to each module's `sourceSets` block
- [ ] Fake implementations use `MutableStateFlow` + helpers (not mock frameworks)
- [ ] ViewModel tests cover: initial Loading, Success with data, each action (success + failure), errorEvents
- [ ] Repository tests cover: Flow emission, write operations, cache-first (if remote exists)
- [ ] All test classes compile (no unresolved references)

### 9. Report

Output a summary listing files created, build configuration changes, test coverage (ViewModel/Repository/UseCase), and next steps (`./gradlew :<feature>:feature:allTests`).

## Troubleshooting

- `commonTest` source set doesn't exist → create the directory structure: `src/commonTest/kotlin/...`
- Test dependencies missing from version catalog → add entries per [test-patterns.md](../../references/test-patterns.md#test-dependencies-buildgradlekts)
- Fake class conflicts with existing test doubles → ask user whether to merge into the existing fake or replace it
