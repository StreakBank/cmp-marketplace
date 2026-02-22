# Plugin Dogfooding Findings

Captured from running cmp-marketplace plugins against [example-cmp](../example-cmp).

---

## Quality Agents

_Quality agents not re-run in this phase. See prior audit conversation for agent results._

---

## Scaffold Skills

### scaffold-feature
- **Feature name:** orders
- **Completed:** yes
- **Compiles?** yes
- **Issues:** None — context detection, package paths, sealed UiState, stateIn(), design tokens, strings.xml all correct

### add-screen
- **Module/screen:** orders / order-details
- **Completed:** yes
- **Compiles?** yes
- **Issues:** None — matches existing detail screen pattern, navigation wiring, DI registration all correct

### add-navigation-tab
- **Feature:** orders
- **Completed:** yes
- **Issues:** None — TopLevelDestination entry, NavHost wiring, icon choice all correct

### add-remote-datasource
- **Module tested:** orders
- **Completed:** yes
- **Compiles?** yes
- **Issues:** None — cache-first pattern, refresh method, Mock impl all correct

### add-ktor-networking
- **Module tested:** products, orders
- **Completed:** yes
- **Compiles?** yes
- **Issues:** None — core/network module created correctly, HttpClient factory, safeApiCall(), DTOs + toDomain(), KtorRemoteDataSource, DI swap, MockEngine for offline dev

### add-room-database
- **Module tested:** products, favorites
- **Completed:** yes (with reference doc fixes)
- **Compiles?** yes (after fixing reference doc patterns)
- **Issues:**
  1. **P0:** Reference doc missing `@ConstructedBy` annotation — iOS KSP fails without it
  2. **P0:** Reference doc expect/actual function pattern broken — Android needs Context but expect has no params
  3. **P1:** Reference doc doesn't mention `startKoin` requirement for `androidContext()`
- **Suggested fixes:** See "Reference File Inaccuracies" section below

### add-image-loading
- **Module tested:** products, orders
- **Completed:** yes
- **Compiles?** yes
- **Issues:** image-loading-patterns.md `platformCacheDirectory` has the same expect/actual Context problem as persistence-patterns.md. Worked around by skipping custom disk cache (Coil handles per-platform caching automatically).

### scaffold-tests
- **Module tested:** cart, favorites, orders, settings
- **Completed:** yes
- **Compiles?** yes
- **Tests pass?** yes — 52 tests across 8 test source sets
- **Issues:**
  1. Settings ViewModel tests needed adjustment for `WhileSubscribed` stateIn timing — must await Loading→Success inside `test {}` block, not outside
  2. Cart's non-suspend LocalDataSource methods required special handling in fakes
- **Suggested fixes:** test-patterns.md should mention `WhileSubscribed` timing pattern

### extract-strings
- **Module tested:** orders
- **Completed:** yes
- **Issues:** Found 3 hardcoded error strings in ViewModels. All UI composable files already properly use `stringResource()`.

### fix-imports
- **Completed:** yes
- **Issues:** All 138 resource imports across 22 files are correct. No fixes needed.

### upgrade-dependencies
- **Completed:** yes
- **Issues:** All versions already current (updated during prior phases)

### polish-ui (v2.1.0)
- **Modules tested:** orders, products, wishlist
- **Completed:** yes
- **Compiles?** YES (after fixing P0 import issue below)

#### Step 1: orders
- **Context detection:** Correct — rootProject.name=Example, package_base=com.example
- **Design Assessment:** Chose dense/monitoring direction — appropriate for order tracking
- **AnimDuration.kt created:** Yes — `core/feature/designsystem/AnimDuration.kt`
- **ShimmerEffect.kt created:** Yes — `core/feature/designsystem/ShimmerEffect.kt`
- **EmptyStateView/ErrorStateView reuse:** Yes — detected and reused existing
- **Shimmer shape match:** Yes — horizontal card shape (image + text lines + trailing badge) for list, two-card layout for details
- **AnimatedContent:** Yes — wraps `when(uiState)` with crossfade on both screens
- **Card recipes:** Kept existing OrderCard (already well-structured HorizontalCard pattern with leading image + status badge)
- **Pull-to-refresh:** Yes — added `PullToRefreshBox` wrapping Success state's LazyColumn
- **Architecture untouched:** Yes — ViewModel, Repository, DI, Navigation files unchanged
- **Token violations found in existing code:** `RoundedCornerShape(Spacing.xs)` in OrderCard → fixed to `Radius.xs`
- **Raw dp/ms/alpha literals:** None in output
- **Issues:**
  1. INFO: Skill correctly identified that landing screen had no Error state (it uses SharedFlow for errors via Snackbar, Empty state for no data)
  2. INFO: Added `key = { it.id }` to LazyColumn items for better diff performance
  3. INFO: Added `Icons.Outlined.Receipt` to empty state (was missing icon)

#### Step 2: products
- **Skipped AnimDuration/ShimmerEffect creation:** Yes — correctly detected they already exist
- **Design Assessment:** Chose visual/spacious direction — different from orders' dense/monitoring. Appropriate for browsing/discovery.
- **Different card recipes:** Yes — grid-based shimmer (matching LazyVerticalGrid) vs orders' list-based shimmer
- **AnimatedContent:** Yes — both ProductListScreen and ProductDetailsScreen
- **Pull-to-refresh:** Yes — added `PullToRefreshBox` on ProductListScreen
- **Filter/Search handling:** Preserved existing SearchBar + CategoryFilterBar inside Success and Empty states
- **Token violations fixed in existing code:**
  1. `320.dp` raw literal in `GridCells.Adaptive` → `Spacing.xxxxl * 5` (320dp = 64*5)
  2. `RoundedCornerShape(Spacing.sm)` in ProductCard → `Radius.sm`
  3. `RoundedCornerShape(Spacing.md)` in ProductDetailsScreen → `Radius.md`
  4. `0.6f` alpha in ProductCard and ProductDetailsScreen → `ContentAlpha.low`
  5. `0.7f` alpha in ProductCard → `ContentAlpha.medium`
- **Shimmer for ProductDetailsScreen:** Hero image + info card + actions card shimmer — matches actual content layout
- **Issues:**
  1. P2: Existing ProductCard had `RoundedCornerShape(Spacing.sm)` — using spacing token for border radius is a pattern the quality agents should flag
  2. P2: `320.dp` → `Spacing.xxxxl * 5` works but is arguably less readable; a dedicated grid token might be better
  3. INFO: `import dp` was correctly removed after eliminating the raw literal

#### Post-dogfooding fix (v2.1.1)
- **Behavioral fix:** polish-ui rewrote to mandate item composable rewrites, staggered list animations, and scale-on-press micro-interactions. Prior behavior was too deferential — kept existing Card composables as-is even when they had no visual hierarchy.
- **P0 fix:** Added explicit `PullToRefreshBox` import path to ui-recipes.md.

#### Step 3: scaffold-feature wishlist → polish-ui wishlist
- **scaffold-feature report:** Did NOT explicitly mention `/cmp-scaffold:polish-ui wishlist` as next step (the skill prompt says to suggest it, but this was a manual run — noting for skill verification)
- **polish-ui on fresh scaffold:**
  - Loading `CircularProgressIndicator` → shimmer skeleton matching item rows
  - `when(uiState)` → `AnimatedContent` with crossfade
  - Empty state: already had `EmptyStateView` with icon (scaffold did this correctly)
  - Pull-to-refresh: Correctly skipped — local-only module with no `refresh()`
  - No new design system files needed (all created in step 1)
- **Issues:**
  1. INFO: Maximum surface area validated — bare scaffold had all 3 anti-patterns (CircularProgressIndicator, bare Text items, no animations)
  2. INFO: Scaffold → polish pipeline works cleanly end-to-end

---

## Cross-Cutting Findings

### Patterns the example uses that the marketplace doesn't cover
- `startKoin` in platform entry points (Android Application + iOS initKoin guard)
- `@ConstructedBy` + `expect object ... : RoomDatabaseConstructor` pattern for Room KMP
- `NavigationSuiteScaffold` composable state hoisting (non-composable builder lambdas)

### Patterns the marketplace enforces that the example violates
- None found — example follows all marketplace patterns correctly

### Missing skills identified during dogfooding
- No `add-theming` skill exists — theming was implemented manually following theming-patterns.md
- No `add-adaptive-layouts` skill exists — adaptive layouts implemented manually
- Both could be useful skills but are infrequent enough that reference docs suffice

### Reference file inaccuracies discovered
1. **persistence-patterns.md** — 3 bugs (see Priority Actions)
2. **adaptive-layouts-patterns.md** — version mismatch between adaptive and navigation suite artifacts
3. **image-loading-patterns.md** — `platformCacheDirectory` expect/actual has same Context problem
4. **ui-recipes.md** — Pull-to-Refresh recipe missing import; `PullToRefreshBox` lives in `androidx.compose.material3.pulltorefresh`, not `androidx.compose.material3`

### General UX / prompt / output format issues
- `WhileSubscribed` stateIn timing in test templates can cause false test failures
- KSP version documentation should clarify KSP 2.x is decoupled from Kotlin versions
- Cart module's non-suspend `CartLocalDataSource` methods are incompatible with Room (future interface cleanup needed)

---

## Priority Actions for Marketplace

| # | Finding | Plugin/File | Severity | Fix |
|---|---------|-------------|----------|-----|
| 1 | Database missing `@ConstructedBy` | persistence-patterns.md | P0 | Add `@ConstructedBy` + `expect object Constructor` to Database template |
| 2 | expect/actual function broken (Context) | persistence-patterns.md | P0 | Replace with expect/actual Koin Module pattern |
| 3 | iOS `instantiateImpl()` deprecated | persistence-patterns.md | P0 | Remove from iOS template, rely on `@ConstructedBy` |
| 4 | Missing `startKoin` pattern | persistence-patterns.md | P1 | Add section on Koin init for Android Context |
| 5 | Version mismatch (adaptive vs nav suite) | adaptive-layouts-patterns.md | P1 | Document separate version entries |
| 6 | `platformCacheDirectory` Context issue | image-loading-patterns.md | P2 | Document simpler approach without custom cache dir |
| 7 | `WhileSubscribed` test timing | test-patterns.md | P2 | Add note about awaiting inside `test {}` block |
| 8 | iOS missing `BundledSQLiteDriver` | persistence-patterns.md | P0 | Add `.setDriver(BundledSQLiteDriver())` to iOS database builder — Room KMP on non-Android requires explicit SQLite driver |
| 9 | ~~`PullToRefreshBox` wrong import path~~ | ui-recipes.md | ~~P0~~ FIXED | ~~Recipe doesn't specify import~~ — Fixed in v2.1.1: added explicit `import androidx.compose.material3.pulltorefresh.PullToRefreshBox` to recipe |

---

## Coverage Summary

| Skill | Phase | Target | Ref Doc Validated | Result |
|-------|-------|--------|-------------------|--------|
| scaffold-feature | 1 | orders | code-templates, project-context, design-tokens, string-resources | PASS |
| add-screen | 1 | orders | code-templates | PASS |
| add-navigation-tab | 1 | orders | — | PASS |
| add-remote-datasource | 1 | orders | data-patterns | PASS |
| add-ktor-networking | 2 | products, orders | networking-patterns | PASS |
| add-room-database | 3 | products, favorites | persistence-patterns | PASS (with fixes) |
| add-image-loading | 4 | products, orders | image-loading-patterns | PASS |
| Manual: theming | 5 | core/feature | theming-patterns | PASS |
| Manual: adaptive | 6 | composeApp, products | adaptive-layouts-patterns | PASS (with fixes) |
| scaffold-tests | 7 | cart, favorites, orders, settings | test-patterns | PASS |
| extract-strings | 7 | orders | string-resources | PASS |
| fix-imports | 7 | all | — | PASS |
| upgrade-dependencies | 7 | all | — | PASS |
| polish-ui | 8 | orders, products, wishlist | ui-recipes, design-tokens | PASS (with fixes) |

**All 13 skills exercised. All reference docs validated. 6 reference doc bugs found and documented.**
