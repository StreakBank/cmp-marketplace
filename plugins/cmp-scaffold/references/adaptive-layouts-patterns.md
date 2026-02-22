# Adaptive Layout Patterns

Responsive layout patterns for Compose Multiplatform using Material 3 Adaptive libraries. All APIs are available in `commonMain`.

---

## Version Catalog Entries

```toml
[versions]
compose-material3-adaptive = "1.3.0-alpha02"                 # Alpha by semver, but stable for production within CMP 1.10+
compose-material3-navigation-suite = "1.10.0-alpha05"        # Navigation suite follows CMP versioning, NOT adaptive versioning. Latest available as of Feb 2026 — check for stable release.

[libraries]
compose-material3-adaptive = { group = "org.jetbrains.compose.material3.adaptive", name = "adaptive", version.ref = "compose-material3-adaptive" }
compose-material3-adaptive-layout = { group = "org.jetbrains.compose.material3.adaptive", name = "adaptive-layout", version.ref = "compose-material3-adaptive" }
compose-material3-adaptive-navigation = { group = "org.jetbrains.compose.material3.adaptive", name = "adaptive-navigation", version.ref = "compose-material3-adaptive" }
compose-material3-navigation-suite = { group = "org.jetbrains.compose.material3", name = "material3-adaptive-navigation-suite", version.ref = "compose-material3-navigation-suite" }
```

> **Important:** `material3-adaptive-navigation-suite` uses CMP-aligned versioning (1.10.x), NOT the same version as the other adaptive libraries (1.3.x). These must be separate version entries.

### build.gradle.kts (commonMain)

```kotlin
commonMain.dependencies {
    implementation(libs.compose.material3.adaptive)
    implementation(libs.compose.material3.adaptive.layout)
    implementation(libs.compose.material3.adaptive.navigation)
    implementation(libs.compose.material3.navigation.suite)
}
```

---

## WindowSizeClass Detection

Use `currentWindowAdaptiveInfo()` to detect window size in commonMain:

```kotlin
import androidx.compose.material3.adaptive.currentWindowAdaptiveInfo
import androidx.compose.material3.adaptive.WindowAdaptiveInfo
import androidx.compose.material3.windowsizeclass.WindowSizeClass

@Composable
fun MyApp() {
    val windowInfo = currentWindowAdaptiveInfo()
    val widthSizeClass = windowInfo.windowSizeClass

    val isCompact = !widthSizeClass.isWidthAtLeastBreakpoint(
        WindowSizeClass.WIDTH_DP_MEDIUM_LOWER_BOUND
    )
    val isExpanded = widthSizeClass.isWidthAtLeastBreakpoint(
        WindowSizeClass.WIDTH_DP_EXPANDED_LOWER_BOUND
    )
}
```

### Size Classes

| Category | Width | Typical Devices |
|----------|-------|-----------------|
| Compact | < 600dp | Phone portrait |
| Medium | 600–839dp | Phone landscape, small tablet |
| Expanded | 840dp+ | Tablet, desktop |

---

## Adaptive Navigation Pattern

Switch between BottomBar (compact), NavigationRail (medium), and NavigationDrawer (expanded) using `NavigationSuiteScaffold`:

```kotlin
import androidx.compose.material3.adaptive.navigationsuite.NavigationSuiteScaffold

@Composable
fun AppContent(
    currentDestination: TopLevelDestination,
    onDestinationSelected: (TopLevelDestination) -> Unit,
) {
    NavigationSuiteScaffold(
        navigationSuiteItems = {
            TopLevelDestination.entries.forEach { destination ->
                item(
                    selected = destination == currentDestination,
                    onClick = { onDestinationSelected(destination) },
                    icon = { destination.Icon(selected = destination == currentDestination) },
                    label = { Text(destination.iconText) },
                )
            }
        },
    ) {
        // Screen content — NavigationSuiteScaffold automatically picks
        // BottomBar, NavigationRail, or NavigationDrawer based on window size
        AppNavHost()
    }
}
```

`NavigationSuiteScaffold` automatically selects the right navigation chrome:
- **Compact width** → Bottom navigation bar
- **Medium width** → Navigation rail
- **Expanded width** → Permanent navigation drawer

> **Important:** The `navigationSuiteItems` lambda is NOT `@Composable`. Any `@Composable` state reads (e.g., `collectAsState()` getters) must be hoisted into the parent composable scope and passed as variables.

---

## Responsive Grid/List

Adapt column count based on window width using `LazyVerticalGrid` with `Adaptive`:

```kotlin
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid

@Composable
fun <Feature>Grid(items: List<<Model>>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        contentPadding = PaddingValues(Spacing.md),
        horizontalArrangement = Arrangement.spacedBy(Spacing.sm),
        verticalArrangement = Arrangement.spacedBy(Spacing.sm),
    ) {
        items(items, key = { it.id }) { item ->
            <Feature>Card(item)
        }
    }
}
```

`GridCells.Adaptive(minSize)` automatically calculates column count from available width — no manual breakpoint logic needed.

---

## List-Detail Pattern

Use `ListDetailPaneScaffold` for tablet/desktop list-detail layouts:

```kotlin
import androidx.compose.material3.adaptive.layout.ListDetailPaneScaffold
import androidx.compose.material3.adaptive.layout.ListDetailPaneScaffoldRole
import androidx.compose.material3.adaptive.navigation.rememberListDetailPaneScaffoldNavigator

@Composable
fun <Feature>ListDetail(items: List<<Model>>) {
    val navigator = rememberListDetailPaneScaffoldNavigator<Long>()

    ListDetailPaneScaffold(
        directive = navigator.scaffoldDirective,
        value = navigator.scaffoldValue,
        listPane = {
            <Feature>List(
                items = items,
                onItemClick = { item ->
                    navigator.navigateTo(
                        pane = ListDetailPaneScaffoldRole.Detail,
                        content = item.id,
                    )
                },
            )
        },
        detailPane = {
            navigator.currentDestination?.content?.let { itemId ->
                <Feature>Detail(itemId = itemId)
            }
        },
    )
}
```

**Behavior:**
- **Compact** — list and detail are full-screen; navigating to detail pushes a new screen
- **Medium/Expanded** — side-by-side list and detail panes

---

## Orientation & State Preservation

### Preserving State Across Configuration Changes

Use `rememberSaveable` for UI state that should survive configuration changes (rotation, window resize):

```kotlin
var selectedTabIndex by rememberSaveable { mutableIntStateOf(0) }
var searchQuery by rememberSaveable { mutableStateOf("") }
```

### Scroll Position

`rememberLazyListState()` automatically preserves scroll position when items provide a `key` parameter. No additional work needed:

```kotlin
val listState = rememberLazyListState()
LazyColumn(state = listState) {
    items(data, key = { it.id }) { item ->
        ItemCard(item)
    }
}
```

### Content Width Constraint

On expanded layouts (tablets, desktop), constrain content width to maintain readability. Long text lines (> ~80 characters) reduce reading speed:

```kotlin
Box(
    modifier = Modifier.fillMaxSize(),
    contentAlignment = Alignment.TopCenter,
) {
    LazyColumn(
        modifier = Modifier.widthIn(max = 640.dp),
        contentPadding = contentPadding,
    ) {
        // Content stays readable even on very wide screens
    }
}
```

---

## Key Rules

- Always derive layout from `currentWindowAdaptiveInfo()` — never hardcode breakpoints in dp
- Prefer `NavigationSuiteScaffold` over manual BottomBar/Rail switching — it handles the logic
- Use `GridCells.Adaptive` for grids — let the framework calculate column count
- Use `ListDetailPaneScaffold` for master-detail — it handles single-pane vs dual-pane automatically
- Test on phone, tablet, and desktop window sizes — each may use a different layout
- All adaptive APIs are in `commonMain` — no `expect`/`actual` needed (CMP 1.10+)
- Preserve state across layout changes — use `rememberSaveable` for UI state, `key` for scroll position
- Constrain content width on expanded layouts — `Modifier.widthIn(max = 640.dp)` prevents overly long text lines
