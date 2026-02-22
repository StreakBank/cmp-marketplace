# Compose Performance: Recomposition & State Reads

Patterns for avoiding unnecessary recomposition in Compose Multiplatform. Focus: ViewModel state emission, `combine` reference caching, and strong skipping mode.

---

## Strong Skipping Mode (Default Since Kotlin 2.0)

Since Kotlin 2.0, the Compose compiler enables **strong skipping** by default. This changes how unstable parameters are handled:

| Mode | Stable params | Unstable params | Lambdas |
|------|---------------|-----------------|---------|
| Normal skipping | `.equals()` | Never skip | Must wrap in `remember` |
| **Strong skipping** | `.equals()` | **`===` (referential)** | **Auto-memoized** |

With strong skipping, composables receiving unstable parameters (e.g., cross-module data classes without `@Immutable`) **can still be skipped** — the compiler uses referential equality (`===`) instead of structural equality (`.equals()`).

**Why this matters for KMP:** Data classes defined in `data/api` modules (pure Kotlin, no Compose) are treated as unstable by the Compose compiler in `feature` modules. Without strong skipping, every composable receiving a `Product`, `Order`, etc. would recompose on every emission. Strong skipping handles this automatically via `===`.

---

## Reference Stability in ViewModel combine Chains

The `combine` operator **caches the latest value** from non-emitting upstream flows, preserving object references. This works with strong skipping's `===` check:

```
favorites changes → combine fires:
  products:      same List instance (didn't re-emit)     → === ✓ skip
  categories:    same Set instance (didn't re-emit)      → === ✓ skip
  currentFilter: same instance                           → === ✓ skip
  favorites:     NEW Set instance (the changed flow)     → === ✗ recompose
```

Inside a `LazyColumn`/`LazyVerticalGrid`, only items whose parameters actually changed (by `===` for unstable types, `.equals()` for stable types) recompose. Everything else skips.

### Don't Break Reference Stability

```kotlin
// BAD: creates a new list every emission — breaks === for strong skipping
products = originalProducts.toList()

// BAD: map creates new instances even if transform is identity-like
products = originalProducts.map { it.copy() }

// GOOD: pass through the same instance
products = originalProducts
```

When you **must** transform data in the ViewModel (mapping, filtering, sorting), the resulting new instances will use `===` and won't skip. For most cases this is fine — the entire list recomposes but individual items may still skip if their specific data is referentially identical.

---

## When You Still Need Stability Annotations

If you can't guarantee reference stability (e.g., you must `.map()` or `.toList()` in every emission), you have three options:

### Option 1: `@Immutable` on Data Classes

Add `compose.runtime` dependency to the data module and annotate classes. Compiler uses `.equals()` instead of `===`:

```kotlin
@Immutable
data class Product(val id: String, val name: String, val price: Double)
```

**Trade-off:** Adds a Compose dependency to a pure Kotlin module.

### Option 2: Stability Configuration File

Declare classes as stable in a config file — no code changes needed:

```
// stability-config.txt
{package_base}.*.data.api.model.*
```

Wire in Gradle:

```kotlin
composeCompiler {
    stabilityConfigurationFile = rootProject.file("stability-config.txt")
}
```

**Trade-off:** Easy to forget to update when adding new types.

### Option 3: `kotlinx-collections-immutable`

Use `ImmutableList`/`ImmutableSet` in UiState. The Compose compiler natively recognizes these as stable:

```kotlin
sealed interface ProductUiState {
    data class Success(
        val products: ImmutableList<Product>,
        val favorites: ImmutableSet<String>,
    ) : ProductUiState
}
```

**Trade-off:** Requires converting at the ViewModel boundary.

### When to Use Each

| Scenario | Recommendation |
|----------|---------------|
| `combine` preserves references (most cases) | No annotation needed — strong skipping handles it |
| Must `.map()`/`.toList()` every emission | Option 2 (stability config) — least invasive |
| Performance-critical lists with frequent updates | Option 3 (immutable collections) — strongest guarantee |
| Single shared data class causing issues | Option 1 (`@Immutable`) — targeted fix |

---

## AnimatedContent contentKey

When `AnimatedContent` uses a sealed UiState as `targetState`, always include `contentKey = { it::class }`:

```kotlin
AnimatedContent(
    targetState = uiState,
    transitionSpec = {
        fadeIn(tween(AnimDuration.medium)) togetherWith
            fadeOut(tween(AnimDuration.medium))
    },
    contentKey = { it::class },
    label = "<feature>StateTransition",
) { state ->
    when (state) { ... }
}
```

Without `contentKey`, `AnimatedContent` uses `.equals()` to detect changes. Any data update within `Success` (e.g., toggling a favorite) triggers the fade transition — causing visible flicker across the entire screen.

---

## Performance Budgets

Target metrics for a polished CMP app:

| Metric | Target | How to Achieve |
|--------|--------|----------------|
| Cold launch | < 2s | Avoid blocking `init {}` in ViewModels; use `stateIn()` with `SharingStarted.WhileSubscribed` |
| Screen transition | ≤ 300ms | Use `AnimDuration.medium` (300ms) max. `AnimDuration.long` (500ms) is for non-blocking background animations only (shimmer, loading indicators). |
| Scroll frame rate | 60 fps | Provide `key` to lazy lists; avoid allocations in item composables |
| Touch response | < 100ms | Use `AnimDuration.short` (150ms) for micro-interactions |
| Skeleton visible | < 200ms | Emit `Loading` state immediately from `stateIn()` `initialValue` |

### Lazy List Keys

Always provide `key` to `LazyColumn` / `LazyVerticalGrid` `items()` calls. Keys enable efficient diffing — without them, the entire list recomposes on any change:

```kotlin
// GOOD: keyed items — only changed items recompose
items(products, key = { it.id }) { product ->
    ProductCard(product)
}

// BAD: unkeyed items — all visible items recompose on any list change
items(products) { product ->
    ProductCard(product)
}
```

### Image Loading Performance

- Use `Modifier.size()` on `AsyncImage` to constrain decode size — prevents oversized bitmaps
- Coil manages memory and disk caches automatically — no manual cache management needed
- For large lists with images, the `key` parameter is critical to prevent re-decoding on scroll

### ViewModel Init

Avoid blocking work in ViewModel `init {}`. Use `stateIn()` which starts collection lazily:

```kotlin
// GOOD: non-blocking, starts when collected
val uiState: StateFlow<UiState> = repository.observe()
    .map { UiState.Success(it) as UiState }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)

// BAD: blocks init, delays ViewModel creation
init {
    viewModelScope.launch {
        repository.observe().collect { _uiState.value = UiState.Success(it) }
    }
}
```

---

## Key Rules

- **Don't break references unnecessarily** — avoid `.toList()`, `.map { it.copy() }`, or other transforms that create new instances when the data hasn't changed
- **Trust strong skipping** — cross-module data classes work without `@Immutable` as long as references are stable
- **Always use `contentKey`** on `AnimatedContent` with sealed UiState (rule PERF-1)
- **Stability annotations are a last resort** — only needed when you must transform data on every emission
- **Provide `key` to lazy lists** — `items(data, key = { it.id })` enables efficient diffing and preserves scroll position
- **Respect performance budgets** — screen transitions ≤ 300ms, touch feedback ≤ 150ms, skeleton visible immediately
- **Respect reduce motion** — animations > 150ms should provide a `snap()` fallback when the system reduce motion preference is enabled (see reduce motion pattern in [ui-recipes-surfaces.md](ui-recipes-surfaces.md))
