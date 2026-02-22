---
name: add-navigation-tab
description: Wire an existing feature module as a top-level navigation destination (bottom bar, rail, or drawer depending on adaptive layout)
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Add Navigation Tab

Wire an existing feature module into the app's bottom navigation bar.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `profile`, `notifications`). User may optionally specify icon name or position.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` and module list. Read `composeApp/build.gradle.kts` → `{package_base}`. Glob for `**/navigation/TopLevelDestination.kt` → `{top_level_dest_path}`. Glob for `**/navigation/*NavHost.kt` → `{nav_host_path}`.

### 2. Validate Module

Check for `<feature>/feature/.../navigation/<Feature>Navigation.kt` containing `@Serializable` route and `NavGraphBuilder.<feature>Graph(...)`. If missing, tell user to run `/cmp-scaffold:scaffold-feature <feature>`.

### 3. Read Reference Files

Read `{top_level_dest_path}` (enum pattern), `{nav_host_path}` (graph wiring), and the feature's navigation file (exact route/function names). Detect whether the app uses `NavigationSuiteScaffold` (adaptive layout) or a manual `NavigationBar`/`Scaffold` pattern — the enum and wiring approach differ.

### 4. Update TopLevelDestination.kt

Add new enum entry matching existing pattern:
```kotlin
<FEATURE>(
    labelRes = Res.string.<feature>_tab_label,
    route = <Feature>GraphRoute::class,
) {
    @Composable
    override fun Icon(selected: Boolean) {
        Icon(
            imageVector = if (selected) Icons.Filled.<IconName> else Icons.Outlined.<IconName>,
            contentDescription = stringResource(Res.string.<feature>_tab_label),
        )
    }
},
```

**Placement:** Before the last entry (SETTINGS stays last). **Icons:** orders→ShoppingBag, profile→Person, notifications→Notifications, messages→Chat, search→Search, analytics→BarChart. If unsure, ask. **Imports:** Add `Icons.Filled.*`, `Icons.Outlined.*`, `<Feature>GraphRoute`, and `stringResource`/`Res` imports.

**Adaptive layout:** If `NavigationSuiteScaffold` is in use, the TopLevelDestination pattern uses `item()` in the `navigationSuiteItems` lambda instead of a manual `NavigationBar`. Match the existing `item(icon = { ... }, label = { ... })` pattern rather than the `NavigationBarItem` pattern shown above.

**String resources:** Add `<feature>_tab_label` to the feature module's `strings.xml` (e.g., `<string name="orders_tab_label">Orders</string>`). All navigation labels and contentDescriptions must use `stringResource()` — never hardcoded strings.

### 5. Update NavHost

Add `<feature>Graph()` before the last graph call (settings last). Wire parameters from the `*Graph()` function signature. Add required imports.

### 6. Verify composeApp Dependencies

Ensure `implementation(project(":<feature>:feature"))` in `composeApp/build.gradle.kts`. Add if missing.

### 7. Report

Output a summary listing files modified and next steps (build/run to verify, customize icon if needed).

### 8. Verify

Before reporting, confirm each item — fix any violations:

- [ ] Enum entry uses `stringResource()` for `labelRes` and `contentDescription`
- [ ] Icon override uses `Icons.Filled.*` (selected) and `Icons.Outlined.*` (unselected)
- [ ] String resource `<feature>_tab_label` added to feature module's `strings.xml`
- [ ] `implementation(project(":<feature>:feature"))` in `composeApp/build.gradle.kts`
- [ ] New graph call added to NavHost before settings graph

## Troubleshooting

- NavHost not found → glob for `*NavHost*.kt` under `composeApp/`, ask user for the correct path
- `TopLevelDestination` not found → glob for `*Destination*.kt` and `*Tab*.kt`, ask user to confirm
- Route name conflicts with existing entry → suggest appending a suffix or ask user to rename
- Feature navigation file missing → tell user to run `/cmp-scaffold:scaffold-feature <feature>` first
