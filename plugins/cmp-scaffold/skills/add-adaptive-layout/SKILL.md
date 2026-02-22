---
name: add-adaptive-layout
description: Upgrade the app shell to adaptive NavigationSuiteScaffold with WindowSizeClass detection for responsive navigation
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Adaptive Layout

Upgrade the app's navigation shell from a fixed `Scaffold` + `NavigationBar` to an adaptive `NavigationSuiteScaffold` that automatically switches between bottom bar, navigation rail, and navigation drawer based on window size.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check Existing Navigation Scaffold

Glob for `**/NavigationSuiteScaffold*`, `**/*AppContent*`, `**/MainScreen*`. If `NavigationSuiteScaffold` is already in use, report what's configured and offer to add only missing pieces (e.g., WindowSizeClass detection, list-detail pattern).

Also check current navigation setup — glob for `**/TopLevelDestination.kt` and `**/*NavHost*.kt` to understand the existing structure.

### 3. Read Reference File

Read [adaptive-layouts-patterns.md](../../references/adaptive-layouts-patterns.md) for all adaptive layout templates: version catalog entries, WindowSizeClass detection, NavigationSuiteScaffold, responsive grid, list-detail pattern, and state preservation.

### 4. Add Version Catalog Entries

Check `gradle/libs.versions.toml` for existing adaptive layout entries. Add missing entries for `compose-material3-adaptive` and `compose-material3-navigation-suite`. Check for the latest stable versions compatible with the project's CMP release.

> **Critical:** `material3-adaptive-navigation-suite` uses **CMP-aligned versioning** (e.g., `1.10.x` for CMP 1.10), NOT the same version as the other adaptive libraries (e.g., `1.3.x`). These MUST be separate version entries. If unsure of versions, read [adaptive-layouts-patterns.md](../../references/adaptive-layouts-patterns.md) for the current recommended versions, then run `/cmp-scaffold:upgrade-dependencies` afterward to verify alignment.

Add library entries for:
- `compose-material3-adaptive`
- `compose-material3-adaptive-layout`
- `compose-material3-adaptive-navigation`
- `compose-material3-navigation-suite`

### 5. Add Dependencies

Add to `composeApp/build.gradle.kts` (or `core/feature/build.gradle.kts` if shared):

```kotlin
commonMain.dependencies {
    implementation(libs.compose.material3.adaptive)
    implementation(libs.compose.material3.adaptive.layout)
    implementation(libs.compose.material3.adaptive.navigation)
    implementation(libs.compose.material3.navigation.suite)
}
```

### 6. Replace Scaffold with NavigationSuiteScaffold

Find the main app shell composable (typically wrapping the NavHost with a `Scaffold` + `NavigationBar`). Replace with `NavigationSuiteScaffold`:

- Move tab items into `navigationSuiteItems` lambda
- Move screen content into the content lambda
- Remove manual `NavigationBar` / `NavigationBarItem` usage

**Important:** The `navigationSuiteItems` lambda is NOT `@Composable`. Any `@Composable` state reads (e.g., `collectAsState()` getters, `stringResource()`) must be hoisted into the parent composable scope and passed as plain values.

### 7. Add WindowSizeClass Detection

Add `currentWindowAdaptiveInfo()` call in the app shell to detect window size. This enables:
- Compact width → bottom navigation bar (automatic from NavigationSuiteScaffold)
- Medium width → navigation rail (automatic)
- Expanded width → permanent navigation drawer (automatic)

The `NavigationSuiteScaffold` handles all switching automatically — no manual breakpoint logic needed.

### 8. Update TopLevelDestination Usage

Ensure `TopLevelDestination` entries are compatible with `NavigationSuiteScaffold`:
- Each entry needs `icon` (composable), `label` (text), `selected` state, and `onClick`
- The `item()` call in `navigationSuiteItems` matches the existing pattern

### 9. Add Responsive Content Patterns (Optional)

If the project has list screens that would benefit from list-detail on wider screens:
- Add `ListDetailPaneScaffold` for tablet/desktop layouts
- Use `GridCells.Adaptive(minSize = 160.dp)` for grids that adapt to width
- Constrain content width on expanded layouts: `Modifier.widthIn(max = 640.dp)`

### 10. State Preservation

Ensure state survives window size changes (rotation, resize):
- Use `rememberSaveable` for selected tab index and other UI state
- Provide `key` to lazy lists for scroll position preservation

### 11. Report

Output a summary listing:
- Files created (if any new files were needed)
- Files modified (app shell composable, build.gradle.kts, libs.versions.toml)
- Navigation modes supported: bottom bar (compact), rail (medium), drawer (expanded)
- Next steps:
  - Test on phone portrait, phone landscape, tablet, and desktop window sizes
  - Consider `ListDetailPaneScaffold` for features with list + detail views
  - `/cmp-quality:review-changes` to validate changes

## Troubleshooting

- `NavigationSuiteScaffold` not found → verify `compose-material3-navigation-suite` dependency is added with CMP-aligned version (1.10.x)
- Crash in `navigationSuiteItems` → check for `@Composable` calls inside the lambda (it is NOT @Composable). Hoist state reads.
- Version conflict → ensure adaptive libs use `1.3.x` version and navigation-suite uses `1.10.x` version (they are different version lines)
- Navigation not switching on tablet → verify `currentWindowAdaptiveInfo()` is being called; the scaffold reads it automatically
- Content hidden behind navigation rail → ensure content uses proper padding from the scaffold (it provides content padding automatically)
