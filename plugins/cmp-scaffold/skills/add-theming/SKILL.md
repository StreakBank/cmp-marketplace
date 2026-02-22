---
name: add-theming
description: Set up Material 3 theming with color scheme, typography, dark mode toggle, and DataStore persistence
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Theming

Set up a complete Material 3 theming system with color schemes, typography, dark/light mode toggle, and persistent theme preference via DataStore.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check Existing Theme

Glob for `**/designsystem/Theme.kt`, `**/designsystem/Color.kt`, `**/designsystem/Type.kt`. If a theme composable already exists, report what's already set up and offer to add only the missing pieces (e.g., dark mode toggle, DataStore persistence).

### 3. Read Reference Files

Read [theming-patterns.md](../../references/theming-patterns.md) for theme composable, color scheme, typography, dynamic color, DataStore persistence, and dark mode templates.

Read [design-tokens.md](../../references/design-tokens.md) for spacing, icon sizing, and animation tokens that should be present alongside theme files.

### 4. Create Color.kt

Create `core/feature/src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/Color.kt` with:

- Light and dark color palette definitions
- `LightColorScheme` and `DarkColorScheme` using `lightColorScheme()` / `darkColorScheme()`
- Follow the template from theming-patterns.md

Use Material 3 default purple palette as a starting point. The user can customize colors later with Material Theme Builder.

### 5. Create Type.kt

Create `core/feature/src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/Type.kt` with:

- `AppTypography` val using the `Typography()` constructor
- Standard Material 3 text styles (displayLarge through labelSmall)

### 6. Create Theme.kt

Create `core/feature/src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/Theme.kt` with:

- `AppTheme` composable accepting `darkTheme: Boolean` parameter
- `isSystemInDarkTheme()` as default value
- `MaterialTheme` wrapping with `colorScheme` and `typography`
- Dynamic color `expect`/`actual` support (Android only, optional fallback)

Create platform files:
- `androidMain/.../designsystem/PlatformColorScheme.android.kt` — dynamic color via `dynamicDarkColorScheme`/`dynamicLightColorScheme` on API 31+
- `iosMain/.../designsystem/PlatformColorScheme.ios.kt` — returns `null` (no dynamic color on iOS)
- `commonMain/.../designsystem/PlatformColorScheme.kt` — `expect fun platformColorScheme(darkTheme: Boolean): ColorScheme?`

### 7. Create ThemeMode Enum

Create `core/feature/src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/ThemeMode.kt`:

```kotlin
enum class ThemeMode { LIGHT, DARK, SYSTEM }
```

### 8. Add DataStore Persistence

Create `ThemePreferences` class following theming-patterns.md DataStore template:

- `themeMode: Flow<ThemeMode>` — observes stored preference
- `suspend fun setThemeMode(mode: ThemeMode)` — persists preference

Create `expect`/`actual` DataStore factory for platform file paths:
- Android: `context.filesDir.resolve("theme_prefs.preferences_pb")`
- iOS: `NSHomeDirectory() + "/Documents/theme_prefs.preferences_pb"`

### 9. Create ThemeViewModel

Create `core/feature/src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/ThemeViewModel.kt`:

- Injects `ThemePreferences`
- Exposes `themeMode: StateFlow<ThemeMode>` via `stateIn()` with `WhileSubscribed(5_000)`
- `fun setThemeMode(mode: ThemeMode)` dispatches via `viewModelScope.launch`

### 10. Wire into App.kt

Update the root `App()` composable to:

- Inject `ThemeViewModel` via `koinViewModel()`
- Collect `themeMode` with `collectAsStateWithLifecycle()`
- Derive `darkTheme` from `ThemeMode` (LIGHT → false, DARK → true, SYSTEM → `isSystemInDarkTheme()`)
- Wrap content in `AppTheme(darkTheme = darkTheme)`

### 11. Create DI Module

Create `themeModule` with:
- `single { createDataStore() }`
- `singleOf(::ThemePreferences)`
- `viewModelOf(::ThemeViewModel)`

Add `themeModule` to `AppModule.kt` `includes(...)`.

### 12. Add Dependencies

Check `composeApp/build.gradle.kts` and `core/feature/build.gradle.kts` for:
- `androidx.datastore:datastore-preferences` (for theme persistence)
- `androidx.lifecycle:lifecycle-viewmodel-compose` (for koinViewModel)

Add version catalog entries if missing.

### 13. Report

Output a summary listing:
- Files created (Color.kt, Type.kt, Theme.kt, ThemeMode.kt, ThemePreferences.kt, ThemeViewModel.kt, platform files)
- Files modified (App.kt, AppModule.kt, build.gradle.kts files)
- Theme capabilities: light/dark toggle, system follow, DataStore persistence, dynamic color (Android)
- Next steps:
  - `/cmp-scaffold:add-adaptive-layout` to add responsive navigation that works with the theme
  - Customize colors with [Material Theme Builder](https://m3.material.io/theme-builder)
  - `/cmp-quality:review-changes` to validate changes

## Troubleshooting

- `isSystemInDarkTheme()` not found → ensure Compose Material3 dependency exists
- DataStore import issues → verify `datastore-preferences` is in version catalog and applied
- Dynamic color crash on older Android → the `Build.VERSION.SDK_INT >= S` check prevents this; verify the guard is present
- Theme not applying → ensure `AppTheme { ... }` wraps all content in `App()`, not just individual screens
