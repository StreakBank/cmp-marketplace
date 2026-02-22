# UI Recipes

Opinionated, copy-paste-ready Compose Multiplatform component recipes. Every recipe uses design tokens, Material 3 theme, and `stringResource()`. All commonMain-compatible.

Recipes are organized into focused modules. Load only the modules relevant to your feature — don't load all of them at once.

| Module | Contents | When to load |
|--------|----------|--------------|
| [ui-recipes-loading.md](ui-recipes-loading.md) | Shimmer, EmptyStateView, ErrorStateView, AnimatedContent, ConnectivityBanner | Always (every polished screen needs state handling) |
| [ui-recipes-cards.md](ui-recipes-cards.md) | ImageHeaderCard, HorizontalCard, StatCard, ActionCard, DetailSection | Feature displays items in cards or has a detail screen |
| [ui-recipes-lists.md](ui-recipes-lists.md) | AnimatedLazyColumn, SwipeToDismiss, StickyHeaders, PullToRefresh, SearchBar, FilterChips | Feature has scrollable lists or search |
| [ui-recipes-forms.md](ui-recipes-forms.md) | ValidatedTextField, PasswordField, FormState ViewModel, form submission with loading | Feature has user input / forms |
| [ui-recipes-surfaces.md](ui-recipes-surfaces.md) | Tonal hierarchy, ScaleOnPress, AnimatedFAB, ReduceMotion, BottomSheet, Dialog, gestures | Always (surfaces + micro-interactions + accessibility) |

---

## Design Philosophy

Before applying recipes, understand the feature's context and commit to a visual direction:

- **Purpose**: What problem does this screen solve? Browsing, creating, monitoring, searching?
- **Content type**: Image-heavy, data-dense, text-focused, action-oriented?
- **Tone**: Choose a direction within Material 3 — elevated and spacious, dense and efficient, visual and immersive, clean and editorial
- **Differentiation**: What makes this feature visually memorable? A well-orchestrated loading state, a satisfying pull-to-refresh, rich card compositions, or bold use of tonal surfaces?

**Execute the chosen direction with precision.** A data-dense dashboard needs tight spacing and efficient cards. A discovery feed needs generous whitespace and hero images. A settings screen needs clean hierarchy and clear grouping. Match implementation to intent — every visual choice should reinforce the feature's purpose.

### Anti-Patterns (CMP "AI Slop")

These patterns signal undesigned output. Replace them:

| Pattern | Problem | Replace with |
|---------|---------|--------------|
| Bare `CircularProgressIndicator()` | Generic, no content hint | Shimmer skeleton matching content shape |
| `Text(error.message)` | No recovery path, visually flat | `ErrorStateView` with retry action |
| `Text("No items")` | No guidance, no affordance | `EmptyStateView` with icon, explanation, and action |
| Plain `LazyColumn` with `Text()` items | No visual hierarchy | Rich cards with image, title, metadata, actions |
| No animation between states | Jarring state switches | `AnimatedContent` or `Crossfade` for UiState transitions |
| Raw `16.dp`, `300`, `0.7f` literals | Unmaintainable, inconsistent | `Spacing.lg`, `AnimDuration.medium`, `ContentAlpha.medium` |
| Same surface color everywhere | Flat, no depth | Tonal surface hierarchy (`surface`, `surfaceVariant`, `surfaceContainerLow`) |
| Single typography style | Monotonous | Typography scale contrast (`headlineSmall` + `bodyMedium` + `labelSmall`) |
| `LinearEasing` in `tween()` | Mechanical, robotic motion | `AnimEasing.standard` (`FastOutSlowInEasing`) |
| No reduce motion fallback | Inaccessible to motion-sensitive users | Conditional animation spec with `snap()` fallback |

### Minimum Visual Upgrade Checklist

Every `polish-ui` run **must** produce all of these changes. If any item is skipped, the polish is incomplete:

1. **Every item composable rewritten** with the best-matching card recipe pattern — even if the existing code already uses a `Card` wrapper
2. **List entry animations** — staggered fade + slideInVertically on every `LazyColumn`/`LazyVerticalGrid`
3. **Scale-on-press** on every interactive card/item composable
4. **At least 3 typography styles per item** — e.g., `titleSmall` (primary), `bodySmall` (secondary), `labelSmall` (metadata)
5. **ContentAlpha applied** — `ContentAlpha.medium` on secondary text, `ContentAlpha.low` on tertiary/metadata
6. **Shimmer loading matching content shape** — list items get row-shaped shimmers, grid items get card-shaped shimmers
7. **AnimatedContent** for UiState transitions with crossfade

---

## Key Rules

- **Design tokens only**: `Spacing.*`, `IconSize.*`, `Radius.*`, `Elevation.*`, `ContentAlpha.*`, `AnimDuration.*`, `AnimEasing.*` — no raw `dp`, `ms`, easing, or `alpha` literals
- **Theme colors only**: `MaterialTheme.colorScheme.*` — no hardcoded `Color(0xFF...)`
- **Theme typography only**: `MaterialTheme.typography.*` — no hardcoded `fontSize`
- **String resources only**: `stringResource(Res.string.*)` — no hardcoded user-visible text
- **commonMain only**: all recipes use Foundation + Material 3 APIs available across targets
- **Accessibility**: every interactive element has `contentDescription` or explicit `null` for decorative elements; minimum `Spacing.xxxl` (48dp) touch targets
- **Animation labels**: every `animateX` and `AnimatedContent` call includes a `label` parameter for debugging
