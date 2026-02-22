# Theming Patterns (Material 3)

Material 3 theming patterns for Compose Multiplatform. Provides color schemes, typography, and theme composable setup.

---

## Theme Location

Place theme files in `core/feature/designsystem/`:

```
core/feature/
└── src/commonMain/kotlin/{package_base_path}/core/feature/designsystem/
    ├── Spacing.kt          # Existing design tokens
    ├── IconSize.kt
    ├── AnimDuration.kt     # Animation timing tokens
    ├── AnimEasing.kt       # Animation easing tokens
    ├── Theme.kt            # App theme composable
    ├── Color.kt            # Color scheme definitions
    └── Type.kt             # Typography scale
```

---

## Color Scheme

```kotlin
package {package_base}.core.feature.designsystem

import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.ui.graphics.Color

// Define color palette
val md_theme_light_primary = Color(0xFF6750A4)
val md_theme_light_onPrimary = Color(0xFFFFFFFF)
val md_theme_light_primaryContainer = Color(0xFFEADDFF)
// ... additional colors

val md_theme_dark_primary = Color(0xFFD0BCFF)
val md_theme_dark_onPrimary = Color(0xFF381E72)
val md_theme_dark_primaryContainer = Color(0xFF4F378B)
// ... additional colors

val LightColorScheme = lightColorScheme(
    primary = md_theme_light_primary,
    onPrimary = md_theme_light_onPrimary,
    primaryContainer = md_theme_light_primaryContainer,
    // ... full scheme
)

val DarkColorScheme = darkColorScheme(
    primary = md_theme_dark_primary,
    onPrimary = md_theme_dark_onPrimary,
    primaryContainer = md_theme_dark_primaryContainer,
    // ... full scheme
)
```

---

## Typography

```kotlin
package {package_base}.core.feature.designsystem

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val AppTypography = Typography(
    displayLarge = TextStyle(fontSize = 57.sp, fontWeight = FontWeight.Normal),
    headlineLarge = TextStyle(fontSize = 32.sp, fontWeight = FontWeight.Normal),
    headlineMedium = TextStyle(fontSize = 28.sp, fontWeight = FontWeight.Normal),
    titleLarge = TextStyle(fontSize = 22.sp, fontWeight = FontWeight.Normal),
    titleMedium = TextStyle(fontSize = 16.sp, fontWeight = FontWeight.Medium),
    bodyLarge = TextStyle(fontSize = 16.sp, fontWeight = FontWeight.Normal),
    bodyMedium = TextStyle(fontSize = 14.sp, fontWeight = FontWeight.Normal),
    labelLarge = TextStyle(fontSize = 14.sp, fontWeight = FontWeight.Medium),
    labelMedium = TextStyle(fontSize = 12.sp, fontWeight = FontWeight.Medium),
)
```

---

## Theme Composable

```kotlin
package {package_base}.core.feature.designsystem

import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}
```

---

## Usage in App Composable

```kotlin
@Composable
fun App() {
    AppTheme {
        // App content — all children inherit the theme
        AppNavHost()
    }
}
```

---

## Dark/Light Toggle (Settings Module)

```kotlin
// In a settings ViewModel or preferences store
enum class ThemeMode { LIGHT, DARK, SYSTEM }

// In App composable
@Composable
fun App(themeMode: ThemeMode = ThemeMode.SYSTEM) {
    val darkTheme = when (themeMode) {
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
    }
    AppTheme(darkTheme = darkTheme) {
        AppNavHost()
    }
}
```

---

## Dark Mode Best Practices

Material 3's tonal surface system handles most dark mode concerns automatically. Follow these guidelines for quality dark themes:

- **No pure black backgrounds** — Material 3 uses tonal surfaces (`surface`, `surfaceContainer`, etc.) that provide subtle dark grays. Never override with `Color.Black` / `Color(0xFF000000)`.
- **Brand color saturation** — dark mode variants should be lighter and less saturated than light mode. Material 3's tonal palette handles this automatically when generated from Material Theme Builder.
- **Surface hierarchy in dark mode** — use tonal lift instead of shadows for depth: `surface` (lowest) → `surfaceContainer` → `surfaceContainerHigh` (highest). Shadows are nearly invisible on dark backgrounds.
- **System bars** — use `enableEdgeToEdge()` in the Activity for edge-to-edge rendering. System bars will be transparent and adapt to the current theme automatically.

```kotlin
// Android Activity — call in onCreate() before setContent()
enableEdgeToEdge()
```

---

## Color Accessibility

### WCAG AA Contrast Minimums

| Element | Minimum Ratio | Notes |
|---------|---------------|-------|
| Body text (< 18sp) | 4.5:1 | Against its background |
| Large text (>= 18sp or 14sp bold) | 3:1 | Headlines, titles |
| UI components (icons, borders) | 3:1 | Against adjacent colors |

Material 3's default palette is AA-compliant when using semantic color pairs (`onSurface` on `surface`, `onPrimary` on `primary`, etc.). Do not mix these pairs.

### Rules

- **Never use `Color(0xFF...)` in composable files** — always use `MaterialTheme.colorScheme.*`. Hardcoded colors bypass theme-aware contrast guarantees. **Exception:** Image overlay scrims (e.g., `Color.Black.copy(alpha = ...)` gradient over photos for text legibility) are acceptable because theme colors cannot guarantee contrast over arbitrary image content.
- **Custom status colors** (success green, warning amber, info blue) must be verified for contrast in **both** light and dark modes. Add them to `LightColorScheme` / `DarkColorScheme` as custom extension properties.
- **Alpha on text** — only use `ContentAlpha.*` tokens. Alpha values below `ContentAlpha.disabled` (0.5f) on text will likely fail WCAG AA in both themes.
- **Tool reference** — use [Material Theme Builder](https://m3.material.io/theme-builder) to generate AA-compliant palettes from a seed color.

---

## Dynamic Color (Android Only, Optional)

```kotlin
// androidMain
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import android.os.Build
import androidx.compose.ui.platform.LocalContext

@Composable
actual fun platformColorScheme(darkTheme: Boolean): ColorScheme? {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val context = LocalContext.current
        if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
    } else null
}

// commonMain
@Composable
expect fun platformColorScheme(darkTheme: Boolean): ColorScheme?

// In AppTheme
val colorScheme = platformColorScheme(darkTheme)
    ?: if (darkTheme) DarkColorScheme else LightColorScheme
```

---

## Theme Persistence (DataStore)

Store the user's theme preference across app restarts using DataStore with `expect`/`actual` for the platform file path.

### Version Catalog Entries

```toml
[versions]
datastore = "1.1.7"

[libraries]
datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
```

### DataStore Setup

```kotlin
// commonMain — core/feature/designsystem/
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey

private val THEME_MODE_KEY = stringPreferencesKey("theme_mode")

class ThemePreferences(private val dataStore: DataStore<Preferences>) {

    val themeMode: Flow<ThemeMode> = dataStore.data.map { prefs ->
        prefs[THEME_MODE_KEY]?.let { ThemeMode.valueOf(it) } ?: ThemeMode.SYSTEM
    }

    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.edit { prefs -> prefs[THEME_MODE_KEY] = mode.name }
    }
}
```

### Platform DataStore Factory

```kotlin
// commonMain
expect fun createDataStore(): DataStore<Preferences>

// androidMain
actual fun createDataStore(): DataStore<Preferences> =
    createDataStore(context.filesDir.resolve("theme_prefs.preferences_pb").absolutePath)

// iosMain
actual fun createDataStore(): DataStore<Preferences> =
    createDataStore(NSHomeDirectory() + "/Documents/theme_prefs.preferences_pb")
```

### DI

```kotlin
val themeModule = module {
    single { createDataStore() }
    singleOf(::ThemePreferences)
    viewModelOf(::ThemeViewModel)
}
```

---

## Theme State in ViewModel

```kotlin
class ThemeViewModel(
    private val themePreferences: ThemePreferences,
) : ViewModel() {

    val themeMode: StateFlow<ThemeMode> = themePreferences.themeMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)

    fun setThemeMode(mode: ThemeMode) {
        viewModelScope.launch { themePreferences.setThemeMode(mode) }
    }
}
```

---

## Wiring in App.kt

```kotlin
@Composable
fun App() {
    val themeViewModel: ThemeViewModel = koinViewModel()
    val themeMode by themeViewModel.themeMode.collectAsStateWithLifecycle()

    val darkTheme = when (themeMode) {
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
    }

    AppTheme(darkTheme = darkTheme) {
        AppNavHost()
    }
}
```

---

## Key Rules

- Always wrap app content in `AppTheme { ... }`
- Use `MaterialTheme.colorScheme.*` for colors — never hardcode `Color(0xFF...)` in composables
- Use `MaterialTheme.typography.*` for text styles — never hardcode `fontSize` in composables
- `isSystemInDarkTheme()` is available in commonMain (CMP provides it)
- Dynamic color is Android-only and optional — requires `expect`/`actual`
- Theme preference persists via DataStore — never store in ViewModel state alone
