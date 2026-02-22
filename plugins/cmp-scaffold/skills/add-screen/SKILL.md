---
name: add-screen
description: Add a new screen to an existing feature module with ViewModel, UiState, navigation, and DI
argument-hint: <module> <screen-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Screen to Feature Module

Add a new screen to an existing feature module. Lighter than `scaffold-feature` — use when a module needs an additional screen.

## Input

`$ARGUMENTS` — `<module> <screen_name>` (e.g., `cart returns`, `products comparison`). First word = module, rest = screen name.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Validate Module Exists

Check for `<module>/feature/build.gradle.kts` and `<module>/feature/src/commonMain/kotlin/{package_base_path}/<module>/feature/`. If missing, tell the user to run `/cmp-scaffold:scaffold-feature <module>` first.

### 3. Read Existing Module Patterns

Read these files to match conventions: existing `*Screen.kt`, `*ViewModel.kt`, `*UiState.kt`, `navigation/*.kt`, `di/*Module.kt`, `composeResources/values/strings.xml`.

### 4. Create Screen Directory

Create `<module>/feature/src/commonMain/kotlin/{package_base_path}/<module>/feature/<screen_name>/`. Multi-word names use lowercase concatenation (e.g., `orderdetails`).

### 5. Create UiState, ViewModel, Screen

Use templates from [code-templates.md](../../references/code-templates.md):
- **UiState** — sealed interface with Loading + Success
- **ViewModel** — `stateIn()` pattern, `errorEvents` SharedFlow
- **Screen** — `koinViewModel()`, `collectAsStateWithLifecycle()`, SnackbarHost, exhaustive `when`. Use design tokens from [design-tokens.md](../../references/design-tokens.md) (no raw dp). Use string resources per [string-resources.md](../../references/string-resources.md) (no hardcoded strings). Provide `key` in `items()` calls for `LazyColumn`/`LazyVerticalGrid`.

### 6. Update Navigation

**Nested graph:** Add `@Serializable` route, `NavGraphBuilder.<screenName>Screen(...)`, `NavController.navigateTo<ScreenName>()`.
**Flat:** Add alongside existing composable.

### 7. Update DI Module

Add `viewModelOf(::<ScreenName>ViewModel)`. Use factory pattern if route parameters are needed.

### 8. Add String Resources

Add entries to module's `strings.xml` per [string-resources.md](../../references/string-resources.md).

### 9. Verify

Before reporting, confirm each item — fix any violations:

- [ ] UiState uses `sealed interface` (not `sealed class`)
- [ ] ViewModel uses `stateIn()` pattern with `WhileSubscribed(5_000)`
- [ ] No raw `dp` literals — all sizing uses `Spacing.*`, `IconSize.*`, `Radius.*`
- [ ] No hardcoded strings — all user-visible text uses `stringResource()`
- [ ] `@Serializable` on navigation route
- [ ] `viewModelOf` registered in feature DI module

### 10. Report

Output a summary listing files created, files modified, the route name and navigate function, and next steps (wire navigation from existing screens, implement business logic).
