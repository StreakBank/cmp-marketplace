# Compose Performance: Recomposition & State Reads

Patterns for avoiding unnecessary recomposition in Compose Multiplatform. Focus: ViewModel state emission, `combine` reference caching, and the strong-skipping stability calculus.

> **Companion docs (same story, audit + fix side).** This file is the authoring-time
> guidance. The `performance-audit` agent (sibling `cmp-quality` plugin) defines the
> report-grounded **finding bar** — when a stability issue is actually real — and
> `compose-recomposition-migration-recipes.md` (same plugin) gives the two
> behavior-preserving remediation recipes (deferred-reads migration + stability-config
> reconciliation). Reach for them when auditing or fixing; the framing below matches
> theirs.

---

## Strong Skipping Mode (Default Since Kotlin 2.0.20)

Since Kotlin 2.0.20, the Compose compiler enables **strong skipping** by default. This changes how parameters gate skipping:

| Mode | Stable params | Unstable params | Lambdas |
|------|---------------|-----------------|---------|
| Normal skipping | `.equals()` | Never skip | Must wrap in `remember` |
| **Strong skipping** | `.equals()` | **`===` (referential)** | **Auto-memoized** |

Two consequences that rewrite the old folklore:

- **Every restartable composable is skippable.** The classic "restartable but not skippable" category is gone — skippability is no longer the discriminator. **Param stability** is.
- **Unstable params still skip — by reference.** A composable receiving an unstable param (e.g. a cross-module data class without `@Immutable`) skips whenever the caller passes a **referentially-identical** (`===`) instance. It only recomposes when the instance identity changes.
- **Lambdas are auto-memoized.** `remember { }` around a lambda passed to a composable is no longer needed for skipping.

**Why this matters for KMP:** Data classes defined in `data/api` modules (pure Kotlin, no Compose) are inferred **unstable** by the Compose compiler in `feature` modules. Under strong skipping that's fine *as long as the instance identity is stable* across recompositions — the `===` check skips them. The trap is the opposite case: a value that is **rebuilt new-but-equal every emission** (the `combine`-built UiState below) fails `===` every time, and only `.equals()`-comparison — which requires a stability annotation — can skip it. So the annotation isn't dead weight; it's the thing standing between you and a whole-subtree recomposition on every state emission.

> **Stability is compiler-inferred — read the report, don't eyeball it.** Whether the
> compiler treats a param as stable or unstable is not reliably visible from source.
> Enable `metricsDestination` / `reportsDestination` on the Compose compiler plugin and
> read `*-composables.txt` (per-param stability) and `*-classes.txt` (per-class
> inference). Gotcha (verify empirically): the metrics options do **not** invalidate
> up-to-date compile tasks, so a warm build emits reports only for modules that happened
> to recompile — force generation with `--rerun-tasks` on the compile task. The
> `performance-audit` agent documents the full evidence step.

---

## Reference Stability in ViewModel combine Chains

`combine` does two things, and the distinction is the whole game:

1. It **caches the latest value** of each non-emitting upstream, preserving those instances' references.
2. Its transform lambda **builds a brand-new output object** — your `UiState` — on every emission. That's a *new-but-equal* instance each time any upstream fires.

```
favorites changes → combine fires:
  products:      same List instance (didn't re-emit)   → reference preserved
  categories:    same Set instance (didn't re-emit)    → reference preserved
  currentFilter: same instance                         → reference preserved
  favorites:     NEW Set instance (the changed flow)   → new reference
  ─────────────────────────────────────────────────────────────────────────
  UiState.Success(...) ← the wrapper combine BUILDS    → NEW instance, EVERY emission
```

The reference caching helps the **fields**, but the **wrapper you emit is new every time.** What happens next depends on how the state reaches the composable:

- **Pass the whole `UiState` wrapper down (the idiomatic, common case).** The wrapper is a new instance every emission, so `===` fails every emission → the receiving composable recomposes **unless the UiState is `@Immutable` / `@Stable`**, which upgrades its comparison to `.equals()`. Because the cached field references make the new wrapper structurally equal to the previous one, `.equals()` returns `true` → the subtree skips. **The annotation is load-bearing here — this is the common case, not an edge case.**
- **Destructure and pass individual fields down.** Non-emitting fields keep their references, so an unstable field skips by `===`; only the field that actually changed recomposes. No wrapper annotation is needed because there's no wrapper on the wire — but this shape is less common, and it multiplies the parameter list.

Inside a `LazyColumn` / `LazyVerticalGrid`, only items whose parameters actually changed (by `===` for unstable types, `.equals()` for stable/annotated types) recompose. Everything else skips.

### Don't Break Reference Stability Gratuitously

```kotlin
// BAD: rebuilds a new instance from an UNCHANGED upstream — breaks === for no reason
products = originalProducts.toList()

// BAD: map creates new element instances even when the transform is identity-like
products = originalProducts.map { it.copy() }

// GOOD: pass the unchanged upstream through untouched
products = originalProducts
```

Reference-breaking a value that *didn't change* is wasteful when it's passed down as an unstable param — it forces a `===` miss for nothing. When you **must** transform (filter, sort, map-to-a-different-type), the new instance is correct and expected; and if it flows into an `@Immutable` UiState compared by `.equals()`, a structurally-equal rebuild still skips. The rule is "don't churn identity for nothing," not "never transform."

---

## Stability Annotations Are Load-Bearing

The old framing — "annotations are a last resort, only for when you must transform on every emission" — is wrong under strong skipping. `@Immutable` / `@Stable` are what let a **new-but-equal** instance skip via `.equals()`, and the `combine`-built UiState above is new-but-equal on **every** emission. Annotate the UiState and per-emission-rebuilt models by default; the annotation is doing real work.

`@Immutable` upgrades a type's comparison from `===` to `.equals()`. Three mechanisms deliver that, chosen by *where the type lives* — not by how desperate you are:

### Option 1: `@Immutable` on module-owned UiState / models (the default)

The feature owns its UiState and screen models, so annotate them at the source:

```kotlin
@Immutable
sealed interface ProductUiState {
    data class Success(
        val products: List<Product>,
        val favorites: Set<String>,
    ) : ProductUiState
}
```

This is the primary tool, not a targeted patch for one problem class — any UiState built in a `combine` / `.map` per emission needs it. `@Stable` is the weaker cousin (it promises "I notify on change") and must **never** sit on a class with mutable public state: that's a false promise the compiler trusts, and the one genuinely-removable stability annotation an audit will flag.

### Option 2: Stability Configuration File (for pure-Kotlin `data/api` types)

A `data/api` type you don't want to give a Compose dependency gets declared stable in a config file — no code change, no Compose import in the pure-Kotlin module:

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

**Trade-off:** the config drifts — entries rot when classes move (dead FQNs the parser silently ignores) and new domain types go unregistered, silently losing `.equals()`-skipping. The report-driven reconciliation recipe (register/annotate + regenerate-to-prove-the-flips + an FQN-existence CI gate) is in `compose-recomposition-migration-recipes.md`.

### Option 3: `kotlinx-collections-immutable` (only for genuine identity churn)

`ImmutableList` / `ImmutableSet` are natively recognized as stable by the Compose compiler:

```kotlin
@Immutable
data class Success(
    val products: ImmutableList<Product>,
    val favorites: ImmutableSet<String>,
)
```

**This is NOT a blanket migration.** An unstable `List` param compares by `===`, which is *correct* whenever the instance is cache-stable across recompositions — and most lists are. Reach for `ImmutableList` only when a list is **genuinely rebuilt with new identity per recomposition** and can't be cached — real, uncacheable identity churn. Sweeping every `List` → `ImmutableList` is cargo-culting: it adds a conversion at the ViewModel boundary for lists that already skipped fine by `===`.

### Generic types follow their type arguments

A registered or annotated container is stable **only if its type arguments are.** `Resource<T>`, `Cached<T>`, `List<T>` are all still **unstable** when `T` is unstable — the compiler propagates argument instability through the container. Registering the container in the stability config (or `@Immutable` on it) does nothing; you must register (or `@Immutable`) the **type argument** too. Never mask this with a star-projection (`Foo<*>`) config entry — that hides which argument is actually unstable instead of fixing it.

### When to Use Each

| Scenario | Recommendation |
|----------|---------------|
| UiState / model rebuilt per emission (combine output, `.map`-per-emission) | **Option 1 `@Immutable`** — the default; load-bearing, not optional |
| Pure-Kotlin `data/api` type used as a composable param | Option 2 (stability config) — no Compose import in the data module |
| A list genuinely rebuilt with new identity every recomposition | Option 3 (immutable collections) — for real identity churn only |
| Unstable param whose instance is cache-stable across recompositions | Nothing — it already skips by `===` |
| Generic container still unstable after registering it | Register/annotate its **type argument**, never `<*>` |

Confirm every stability decision against the compiler report — the proof is "the param flipped to `stable` in the regenerated `*-composables.txt`," not "I added the annotation."

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

- **Stability annotations are load-bearing, not a last resort** — `@Immutable` / `@Stable` on a UiState/model rebuilt new-but-equal per emission (the `combine` case) is what enables `.equals()`-skipping; removing it re-breaks skipping.
- **Unstable ≠ non-skippable** — under strong skipping an unstable param skips by `===` whenever its instance identity is stable; only a new-but-equal instance needs the annotation to skip by `.equals()`.
- **Don't churn identity for nothing** — avoid `.toList()` / `.map { it.copy() }` on *unchanged* upstreams (breaks `===` for free); transform freely when the data actually changed.
- **`ImmutableList` is not a blanket migration** — use it only for lists genuinely rebuilt with new identity per recomposition; a cache-stable `List` already skips by `===`.
- **Generic stability follows type arguments** — register/annotate the type arguments (never a `<*>` mask); a stable container over an unstable `T` is still unstable.
- **The compiler report is the evidence** — enable `metricsDestination` / `reportsDestination`, force with `--rerun-tasks`, and confirm params read `stable` before trusting a stability decision.
- **Lambdas are auto-memoized** — no `remember { }` needed around a lambda param for skipping.
- **Always use `contentKey`** on `AnimatedContent` with sealed UiState (rule PERF-1)
- **Provide `key` to lazy lists** — `items(data, key = { it.id })` enables efficient diffing and preserves scroll position
- **Respect performance budgets** — screen transitions ≤ 300ms, touch feedback ≤ 150ms, skeleton visible immediately
- **Respect reduce motion** — animations > 150ms should provide a `snap()` fallback when the system reduce motion preference is enabled (see reduce motion pattern in [ui-recipes-surfaces.md](ui-recipes-surfaces.md))
