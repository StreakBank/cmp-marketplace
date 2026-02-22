---
name: add-deep-linking
description: Add deep link support to an existing feature's navigation routes for handling custom scheme and https URIs
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Deep Linking

Add deep link support to an existing feature's navigation routes. See [deep-linking-patterns.md](../../references/deep-linking-patterns.md) for templates.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `profile`, `catalog`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`). Derive `{app_scheme}` from `rootProject.name` lowercased with non-alphanumeric characters stripped (e.g., `Example-CMP` → `examplecmp`).

### 2. Validate Feature Module

Glob for `**/<feature>/feature/**/navigation/<Feature>Navigation.kt`. Verify it contains `@Serializable` routes and a `NavGraphBuilder.<feature>Graph(...)` function. If missing, tell the user to run `/cmp-scaffold:scaffold-feature <feature>` first.

### 3. Read Existing Navigation

Read the feature's `<Feature>Navigation.kt` to understand:
- All `@Serializable` route classes/objects (e.g., `<Feature>Route`, `<Feature>DetailRoute`)
- Which routes accept parameters (these are deep link targets)
- The `NavGraphBuilder.<feature>Graph(...)` function structure

### 4. Add Deep Links to Composable Routes

Edit the `<feature>Graph()` function to add `deepLinks` parameter to each `composable<Route>` that should be deep-linkable:

- **Routes with parameters** (e.g., `<Feature>DetailRoute(val id: String)`): Add `navDeepLink<Route>(basePath = "{app_scheme}://<feature>")`. The `@Serializable` fields auto-map to URI path segments.
- **Routes without parameters** (e.g., `<Feature>Route`): Add `navDeepLink<Route>(basePath = "{app_scheme}://<feature>")` if the feature should be directly openable.

Add required imports: `androidx.navigation.navDeepLink`.

### 5. Validate Deep Link Arguments

If the route has parameters, check that the corresponding ViewModel validates the deep link argument (blank/invalid ID handling). If it does not, add a guard at the top of the state flow. See [deep-linking-patterns.md — Argument Validation](../../references/deep-linking-patterns.md) for the pattern.

### 6. Update AndroidManifest.xml

Glob for `**/AndroidManifest.xml` under `composeApp/`. Add an `<intent-filter>` inside the main `<activity>` for the custom scheme:

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="{app_scheme}" android:host="<feature>" />
</intent-filter>
```

If the activity does not already have `android:launchMode="singleTask"`, add it to prevent duplicate instances.

### 7. Add Error Strings

Add deep link error strings to the feature's `strings.xml` (e.g., `<feature>_error_invalid_id`, `<feature>_error_not_found`). Error messages displayed in UI must use `stringResource()`.

### 8. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `navDeepLink<Route>(basePath = ...)` used (NOT manual `uriPattern` string)
- [ ] `@Serializable` route fields auto-map to URI segments — no manual path construction
- [ ] Deep link arguments validated in ViewModel (blank/invalid ID → error state)
- [ ] AndroidManifest has `<intent-filter>` with correct scheme and host
- [ ] Activity has `android:launchMode="singleTask"`
- [ ] Error messages use `stringResource()` — no hardcoded strings
- [ ] `navDeepLink` import added to navigation file

### 9. Report

Output a summary listing files modified and next steps:

- Test with `adb shell am start -a android.intent.action.VIEW -d "{app_scheme}://<feature>/<test-id>" {package_base}.app`
- Test on iOS simulator with `xcrun simctl openurl booted "{app_scheme}://<feature>/<test-id>"`
- Configure iOS URL Types in `Info.plist` for custom scheme support (see deep-linking-patterns.md)
- Add `https://` App Links intent-filter + assetlinks.json when domain verification is ready
- Add Universal Links in Xcode Associated Domains when domain verification is ready
- `/cmp-scaffold:scaffold-tests <feature>` to add tests for deep link argument validation

## Troubleshooting

- Navigation file not found → glob for `*Navigation*.kt` under the feature module, ask user for correct path
- Route has no parameters → deep link still works for opening the screen, but there is nothing to validate
- Multiple deep-linkable routes in one feature → add a separate `navDeepLink` + intent-filter host per route (e.g., `<feature>`, `<feature>-detail`)
- `launchMode="singleTask"` conflicts with existing config → check for `singleTop` and discuss trade-offs with user
- iOS deep links not arriving → remind user that Nav 2.x `handleDeepLink()` is Android-only; iOS requires a platform bridge (see deep-linking-patterns.md)
