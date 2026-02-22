# Design Tokens

All spacing, sizing, elevation, corner radius, and content alpha values must use design tokens. **No raw `dp` literals** in composable files.

## Token Catalogs

### Spacing

| Token | Value | Usage |
|-------|-------|-------|
| `Spacing.xxs` | 2.dp | Stroke widths, minimal gaps |
| `Spacing.xs` | 4.dp | Tight spacing (between label and content) |
| `Spacing.sm` | 8.dp | Small gaps, chip spacing, compact padding |
| `Spacing.md` | 12.dp | Medium gaps, list item spacing |
| `Spacing.lg` | 16.dp | Standard padding, section spacing |
| `Spacing.xl` | 24.dp | Large gaps, section separators |
| `Spacing.xxl` | 32.dp | Empty state padding, major sections |
| `Spacing.xxxl` | 48.dp | Large layout spacing, min touch target |
| `Spacing.xxxxl` | 64.dp | Extra large layout spacing |

### IconSize

| Token | Value | Usage |
|-------|-------|-------|
| `IconSize.sm` | 16.dp | Inline indicators, progress spinners |
| `IconSize.md` | 20.dp | Small toolbar icons, search indicators |
| `IconSize.lg` | 24.dp | Standard icons (Material default) |
| `IconSize.xl` | 48.dp | Medium emphasis icons |
| `IconSize.xxl` | 64.dp | Empty state icons, large indicators |
| `IconSize.xxxl` | 80.dp | Image placeholders, hero icons |

### ContentAlpha

| Token | Value | Usage |
|-------|-------|-------|
| `ContentAlpha.high` | 1.0f | Primary text, active icons |
| `ContentAlpha.medium` | 0.7f | Secondary content (subtitles, descriptions) |
| `ContentAlpha.low` | 0.6f | Tertiary content (captions, metadata) |
| `ContentAlpha.disabled` | 0.5f | Disabled content, hints |

Use with: `color.copy(alpha = ContentAlpha.medium)` — never raw `0.7f` or similar.

### Elevation

| Token | Value | Usage |
|-------|-------|-------|
| `Elevation.none` | 0.dp | Flat surfaces |
| `Elevation.sm` | 1.dp | Subtle lift (cards, list items) |
| `Elevation.md` | 2.dp | Default card elevation |
| `Elevation.lg` | 4.dp | Raised elements (FABs, dialogs) |
| `Elevation.xl` | 8.dp | Prominent elements (navigation drawers) |
| `Elevation.xxl` | 16.dp | Maximum elevation (modals, overlays) |

### Radius (Corner Radius)

| Token | Value | Usage |
|-------|-------|-------|
| `Radius.xs` | 4.dp | Minimal rounding, subtle softening |
| `Radius.sm` | 8.dp | Small cards, chips |
| `Radius.md` | 12.dp | Default cards, buttons |
| `Radius.lg` | 16.dp | Prominent containers, dialogs |
| `Radius.xl` | 24.dp | Large containers, bottom sheets |
| `Radius.full` | 999.dp | Pill shapes, circular elements |

### TouchTarget

| Token | Value | Usage |
|-------|-------|-------|
| `Spacing.xxxl` | 48.dp | Minimum touch target size for accessibility compliance |

All interactive elements (buttons, icon buttons, clickable rows) must have a minimum touch target of 48dp. `IconButton` enforces this automatically. For custom clickable elements, use `Modifier.sizeIn(minWidth = Spacing.xxxl, minHeight = Spacing.xxxl)`.

### AnimDuration

| Token | Value | Usage |
|-------|-------|-------|
| `AnimDuration.short` | 150 | Micro-interactions, ripples, scale-on-press |
| `AnimDuration.medium` | 300 | State transitions, expand/collapse, crossfade (max for screen transitions) |
| `AnimDuration.long` | 500 | Non-blocking background animations only: shimmer loops, loading indicators. NOT for screen/state transitions. |

Values in milliseconds. Defined as `const val` (not `Dp`). Use with `tween(durationMillis = AnimDuration.medium)`. Screen transitions must use `AnimDuration.medium` (300ms) or less to meet performance budgets (see [compose-performance.md](compose-performance.md)).

### AnimEasing

| Token | Value | Usage |
|-------|-------|-------|
| `AnimEasing.standard` | `FastOutSlowInEasing` | Default transitions, state changes |
| `AnimEasing.decelerate` | `LinearOutSlowInEasing` | Elements entering the screen |
| `AnimEasing.accelerate` | `FastOutLinearInEasing` | Elements leaving the screen |
| `AnimEasing.spring` | `spring(dampingRatio = 0.75f, stiffness = StiffnessMedium)` | Interactive feedback, bouncy animations |

Use with `tween(durationMillis = AnimDuration.medium, easing = AnimEasing.standard)`. `LinearEasing` is a violation in UI animation code — always use a named easing token.

## Import

```kotlin
import {package_base}.core.feature.designsystem.Spacing
import {package_base}.core.feature.designsystem.IconSize
import {package_base}.core.feature.designsystem.ContentAlpha
import {package_base}.core.feature.designsystem.Elevation
import {package_base}.core.feature.designsystem.Radius
import {package_base}.core.feature.designsystem.AnimDuration
import {package_base}.core.feature.designsystem.AnimEasing
```

## Rule

Any raw `dp` literal (e.g., `16.dp`, `8.dp`), raw animation duration (e.g., `300`, `tween(150)`), or raw easing (e.g., `LinearEasing`, `tween()` without easing parameter) in composable files is a violation. **Exceptions:** design token definition files (`core/feature/designsystem/*.kt`), and layout constraint breakpoints (`widthIn(max = ...)`, `GridCells.Adaptive(minSize = ...)`) where values are structural layout parameters rather than visual spacing.
