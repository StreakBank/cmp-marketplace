---
name: polish-ui
description: Enhance a feature's UI with modern visual design — rich cards, shimmer loading, animations, and visual hierarchy. Creates shared design system utilities (ShimmerEffect, ScaleOnPress, EmptyStateView, ErrorStateView) if missing.
argument-hint: <feature-name>
context: fork
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Polish Feature UI

> **Don't stack this with `cmp-design-bridge`'s `design-transform`.** The two skills hold opposite theories of visual authority: `polish-ui` *invents* visual treatment from recipes, `design-transform` *reproduces* an already-authored design frame. Never run both on the same screen. When a design frame (e.g. a pulled Claude Design canvas frame) is the source of truth for a screen, `design-transform` owns that screen's visuals — skip `polish-ui` for it, and reserve `polish-ui` for screens with no authoritative design frame.

Upgrade a feature's UI from structurally correct to visually polished. Architecture-preserving — only touches Screen composables, string resources, and design system utilities. **Never modifies ViewModel, Repository, DI, or Navigation.**

**What "polish" means:** This skill must produce visible, dramatic visual improvements. If the UI looks the same after running polish-ui, the skill has failed. At minimum, every run must: rewrite item composables with richer card compositions, add list entry animations, add interactive micro-interactions, and establish typographic hierarchy. The mechanical upgrades (shimmer, AnimatedContent, PullToRefreshBox) are necessary but insufficient.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `profile`, `notifications`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Read Feature Code

Glob for **all** `.kt` files under the feature module's source tree:

```
<feature>/feature/src/commonMain/kotlin/**/*.kt
```

This captures Screen files AND item composables that may live in `views/`, `components/`, or `ui/` subdirectories. Follow imports from Screen files — if a Screen file imports `OrderCard`, `ProductCard`, `WishlistItemRow`, etc., those files **must** be read and rewritten.

Also read:

- `<feature>/data/api/src/commonMain/kotlin/.../model/` — data models (to know available fields)

Understand what the UI currently renders, what item composables exist, and what data fields are available for richer layouts.

### 3. Design Assessment

Assess the feature's content type (list-based, form-based, detail-based, dashboard) and select matching recipes from the reference files. Not all recipes apply — a settings screen doesn't need shimmer; a dashboard doesn't need hero cards.

### 4. Read Design References

Read [ui-recipes.md](../../references/ui-recipes.md) for the design philosophy, anti-patterns, and minimum checklist. Then load **only the recipe modules that match** Step 3's design assessment. **Do NOT load recipe modules that don't apply** — a form-only screen should not load cards or lists recipes.

- **Always load:** [ui-recipes-loading.md](../../references/ui-recipes-loading.md) (shimmer, empty/error states, transitions) and [ui-recipes-surfaces.md](../../references/ui-recipes-surfaces.md) (tonal hierarchy, micro-interactions, reduce motion)
- **Only if feature has item lists:** [ui-recipes-lists.md](../../references/ui-recipes-lists.md) (animated lists, search, pull-to-refresh)
- **Only if feature displays cards/details:** [ui-recipes-cards.md](../../references/ui-recipes-cards.md) (card compositions, detail screens)
- **Only if feature has forms/input:** [ui-recipes-forms.md](../../references/ui-recipes-forms.md) (validated fields, form state)

Also read [design-tokens.md](../../references/design-tokens.md) for token catalogs and [compose-performance.md](../../references/compose-performance.md) for AnimatedContent contentKey, lazy list keys, and animation performance budgets.

### 5. Ensure Design System Utilities

Check that shared utilities exist. Create if missing:

- `core/feature/designsystem/AnimDuration.kt` — animation timing tokens
- `core/feature/designsystem/ShimmerEffect.kt` — `Modifier.shimmer()` extension
- `core/feature/ui/EmptyStateView.kt` — structured empty state composable
- `core/feature/ui/ErrorStateView.kt` — recoverable error view composable

Use the templates from [ui-recipes.md](../../references/ui-recipes.md).

### 6. Upgrade Loading State

Replace `CircularProgressIndicator` (or similar bare loading) with a shimmer skeleton that matches the feature's content shape:

- List features → `ShimmerItem()` rows
- Grid features → shimmer grid cells
- Detail features → shimmer header + body blocks

### 7. Upgrade Empty State

Replace bare empty-list text with `EmptyStateView`:

- Choose an icon that represents the feature's content
- Write a title explaining what's missing
- Write a subtitle with guidance
- Add an action button if the user can create content

### 8. Upgrade Error State

Replace bare error text with `ErrorStateView`:

- Pass the error message
- Wire `onRetry` to the ViewModel's appropriate reload/refresh action

### 9. Add State Transitions

Wrap the `when(uiState)` block in `AnimatedContent` with a crossfade transition using `AnimDuration.medium`. Include `contentKey = { it::class }` so the transition only animates on structural state changes (Loading → Success → Error), not on data changes within the same state type. Provide a descriptive `label`.

### 10. Add List Item Animations

Replace plain `LazyColumn { items(...) }` with the staggered entry animation pattern from ui-recipes.md: wrap each item in `AnimatedVisibility` with `fadeIn` + `slideInVertically` using staggered delays (`index * 50` ms). For grids, use the `animateItem()` modifier. This is the single most impactful visual change — items cascading in vs appearing all at once. Do NOT skip this step.

### 11. Rewrite Item Composables

**REWRITE every item composable.** This is the core visual change — not optional. Read the existing item composable, then rewrite it using the best-matching card recipe from ui-recipes.md as a template. Even if the item already uses a `Card` wrapper, apply the full recipe pattern: structured typography hierarchy (`titleSmall` + `bodySmall` + `labelSmall`), `ContentAlpha` for secondary text, proper spacing tokens, tonal surface treatment. The existing layout is a starting point, not the final product.

Selection guide:

- Model has image URL → use `ImageHeaderCard` or `HorizontalCard` recipe
- Model has numeric value + label → use `StatCard` recipe
- Model has removable/actionable items → use `ActionCard` or add swipe-to-dismiss
- Default → `ElevatedCard` with rich typography hierarchy

Follow imports from the Screen file into `views/` subdirectories. The item composable may live in a separate file — that file **must** be rewritten.

### 12. Establish Visual Hierarchy

Concrete actions (all mandatory):

1. Every item composable must use at least 3 distinct typography styles (e.g., `titleSmall` for primary, `bodySmall` for secondary, `labelSmall` for metadata/timestamps)
2. All secondary text must use `ContentAlpha.medium`, all tertiary text (timestamps, metadata) must use `ContentAlpha.low`
3. If the screen has distinct content sections, wrap them in `Surface(color = surfaceContainerLow)` with `RoundedCornerShape(Radius.lg)`

### 13. Add Micro-Interactions

Add `.scaleOnPress()` modifier to every interactive card/item composable. Create `core/feature/designsystem/ScaleOnPress.kt` if it doesn't exist, using the recipe from ui-recipes.md. This provides tactile feedback that makes the app feel responsive.

### 14. Add String Resources

Add new string resources for:

- Empty state title, subtitle, and action label
- Retry button label (if not already in core strings)
- Any new UI text introduced by the polish

### 15. Verify

Confirm the following — fix any violations:

- [ ] No raw `dp` literals — all sizing uses `Spacing.*`, `IconSize.*`, `Radius.*`, `Elevation.*`
- [ ] No raw millisecond literals — all timing uses `AnimDuration.*`
- [ ] No hardcoded strings — all user-visible text uses `stringResource()`
- [ ] Architecture untouched — ViewModel, Repository, DI, and Navigation files unchanged
- [ ] All `animateX` and `AnimatedContent` calls include `label` parameter
- [ ] Accessibility — `contentDescription` on all icons (or explicit `null` for decorative)
- [ ] Reduce motion — custom animations have accessibility fallback
- [ ] Easing — all `tween()` calls use named easing from `AnimEasing.*` (not `LinearEasing` or default)

### 16. Report

Output a summary listing:

- Design direction chosen and why
- Files created (design system utilities)
- Files modified (screen composables, strings.xml)
- Changes made (loading → shimmer, error → ErrorStateView, etc.)
- Architecture verification: "ViewModel/Repository/DI/Navigation unchanged"
- Next steps: suggest `/cmp-scaffold:add-image-loading` if image fields exist but Coil isn't set up, or `/cmp-quality:review-changes` to validate

## Optional Enhancements

Apply these only when the feature meets the gating condition:

### Pull-to-Refresh
**When:** ViewModel exposes a `refresh()` function or data comes from a remote source.
Wrap content in `PullToRefreshBox`.

### Reduce Motion
**When:** Feature has custom animations > `AnimDuration.short` (150ms).
Add `rememberReduceMotion()` expect/actual. Use `snap()` fallback when enabled. Replace shimmer with static placeholder.

### Connectivity Awareness
**When:** Feature uses remote data (has `refresh()` or remote data source).
Add `ConnectivityBanner` below TopAppBar. Surface `isOffline` and `lastUpdated` from ViewModel.

## Troubleshooting

- Feature module not found → glob for `*/<feature>/feature/` and ask user to confirm the module path
- No `when(uiState)` found → the screen may use a different state pattern; read the screen code and adapt
- `EmptyStateView`/`ErrorStateView` already exist in `core/feature/ui/` → reuse, don't recreate
- ViewModel has no `refresh()` → skip pull-to-refresh; loading shimmer and error retry still apply
- Model has no image URLs → use text-based card compositions; skip image loading recipes
