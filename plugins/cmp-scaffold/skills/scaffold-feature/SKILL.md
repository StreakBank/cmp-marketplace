---
name: scaffold-feature
description: Generate a complete feature module following the established architecture patterns
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Scaffold Feature Module

Generate a complete feature module following established KMP architecture patterns.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `profile`, `notifications`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → extract `rootProject.name` → lowercase = `{resource_prefix}`. Parse `include(...)` lines for existing modules. Read `composeApp/build.gradle.kts` → `namespace` → derive `{package_base}` (strip trailing `.app`). Derive `{package_base_path}` (dots → `/`). Glob for `**/di/AppModule.kt` and `**/navigation/*NavHost.kt` under `composeApp/`.

### 2. Read Local Patterns

Read `settings.gradle.kts` for module registration format. Read an existing feature module's `build.gradle.kts` files to match build configuration patterns.

### 3. Create Module Structure

Create the directory structure from [code-templates.md — Directory Structure](../../references/code-templates.md#directory-structure).

### 4. Data Layer

Create model, repository interface, local data source, repository implementation, and DI module using templates from [code-templates.md](../../references/code-templates.md). Key rules:
- Local data source: interface + InMemory implementation in the **same file**
- DI: `singleOf(::Impl) bind Interface::class`
- Repository reads return `Flow<T>`, writes return `Result<Unit>`

### 5. Feature Layer

Create UiState, ViewModel, Screen, Navigation, and DI module using templates from [code-templates.md](../../references/code-templates.md). Key rules:
- UiState: `sealed interface` (NOT sealed class)
- ViewModel: `stateIn()` pattern (NOT collect in init), `messages: Flow<UiMessage>` one-shot `Channel` (NOT a `MutableSharedFlow<String>` — see [code-templates.md](../../references/code-templates.md#uimessage-one-shot-severity-tagged-message))
- Screen: `koinViewModel()`, `collectAsStateWithLifecycle()`, SnackbarHost consuming `messages`
- Screen: use design tokens from [design-tokens.md](../../references/design-tokens.md) — no raw dp values
- Screen: provide `key` parameter in `LazyColumn`/`LazyVerticalGrid` `items()` calls (e.g., `items(data, key = { it.id })`)
- Navigation: `@Serializable` type-safe routes (Nav 2.x)

### 6. String Resources

Create `strings.xml` following [string-resources.md](../../references/string-resources.md). Convention: `<feature>_<context>_<purpose>`.

### 7. Build Configuration

Follow patterns from existing modules. Feature modules need: `compose.components.resources`, `project(":core:feature")`, `project(":<feature>:data:api")`.

### 8. Registration

- **settings.gradle.kts** — `include(":<feature>:data:api")`, `include(":<feature>:data:impl")`, `include(":<feature>:feature")`
- **composeApp/build.gradle.kts** — `implementation(project(...))` for all sub-modules
- **AppModule.kt** — `includes(<feature>DataModule, <feature>FeatureModule)`

### 9. Navigation Integration

If this is a top-level tab, tell the user to run `/cmp-scaffold:add-navigation-tab <feature>`. Do NOT modify NavHost or TopLevelDestination yourself.

### 10. Report

Output a summary listing files created, files modified, and registration changes. Remind the user about add-navigation-tab if applicable. Suggest next steps:
- `/cmp-scaffold:add-screen <feature> <screen-name>` to add more screens to this module
- `/cmp-scaffold:add-remote-datasource <feature>` to upgrade from local-only to local+remote
- `/cmp-scaffold:scaffold-tests <feature>` to generate unit test scaffolding
- `/cmp-scaffold:polish-ui <feature>` to upgrade loading, error, and empty states with rich visual design

### Verify

Before reporting, confirm each item — fix any violations:

- [ ] Module appears in `settings.gradle.kts`
- [ ] Module registered in `AppModule.kt` DI
- [ ] `composeApp/build.gradle.kts` has module dependency
- [ ] Sealed interface UiState (NOT sealed class)
- [ ] ViewModel uses `stateIn()` pattern (NOT collect in init)
- [ ] Interface-based data sources with `bind` in DI
- [ ] Design tokens only — no raw `dp` literals
- [ ] String resources only — no hardcoded user-visible text
- [ ] `{resource_prefix}` import prefix on generated resources
- [ ] `@Serializable` on navigation routes
- [ ] `key` parameter in lazy list `items()` calls

## Troubleshooting

- Module already exists in `settings.gradle.kts` → ask user if they want to overwrite or skip
- `AppModule.kt` not found → glob for alternative DI wiring files (`*Module.kt` containing `includes(`), ask user to confirm
- Build config pattern can't be matched from existing modules → fall back to [code-templates.md](../../references/code-templates.md) defaults
- `rootProject.name` missing or blank → ask user for the resource prefix to use
