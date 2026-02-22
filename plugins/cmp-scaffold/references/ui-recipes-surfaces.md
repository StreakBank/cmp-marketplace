# UI Recipes: Surfaces, Micro-Interactions & Accessibility

Copy-paste-ready patterns for surface hierarchy, interactive feedback, reduce motion, bottom sheets, and gestures. All commonMain-compatible, design-token-compliant.

---

## Tonal Surface Hierarchy

Use Material 3 tonal elevation to create visual depth without shadows:

| Level | Container | Usage |
|-------|-----------|-------|
| Background | `MaterialTheme.colorScheme.surface` | Page background |
| Raised | `surfaceContainerLow` | Cards, sections above background |
| Emphasized | `surfaceContainerHigh` | Selected items, active surfaces |
| Overlay | `surfaceContainerHighest` | Dialogs, bottom sheets |

```kotlin
Surface(
    color = MaterialTheme.colorScheme.surfaceContainerLow,
    shape = RoundedCornerShape(Radius.lg),
    modifier = Modifier.fillMaxWidth(),
) {
    Column(modifier = Modifier.padding(Spacing.lg)) {
        // Section content
    }
}
```

---

## Micro-Interactions

### Scale-on-Press Modifier

```kotlin
private const val ScaleOnPressFactor = 0.96f

fun Modifier.scaleOnPress(): Modifier = composed {
    var pressed by remember { mutableStateOf(false) }
    val scale by animateFloatAsState(
        targetValue = if (pressed) ScaleOnPressFactor else 1f,
        animationSpec = tween(AnimDuration.short),
        label = "scaleOnPress",
    )
    this
        .graphicsLayer { scaleX = scale; scaleY = scale }
        .pointerInput(Unit) {
            awaitEachGesture {
                awaitFirstDown(requireUnconsumed = false)
                pressed = true
                waitForUpOrCancellation()
                pressed = false
            }
        }
}
```

### Animated FAB (Expand/Collapse on Scroll)

```kotlin
@Composable
fun AnimatedExtendedFab(
    text: String,
    icon: ImageVector,
    onClick: () -> Unit,
    expanded: Boolean,
    modifier: Modifier = Modifier,
) {
    ExtendedFloatingActionButton(
        onClick = onClick,
        expanded = expanded,
        icon = { Icon(icon, contentDescription = null) },
        text = { Text(text) },
        modifier = modifier,
    )
}

// Usage with LazyListState:
val listState = rememberLazyListState()
val fabExpanded by remember {
    derivedStateOf { listState.firstVisibleItemIndex == 0 }
}
```

---

## Reduce Motion Support

Respect the system accessibility setting for users with motion sensitivity.

### Platform Detection

```kotlin
// commonMain
@Composable
expect fun rememberReduceMotion(): Boolean

// androidMain
@Composable
actual fun rememberReduceMotion(): Boolean {
    val context = LocalContext.current
    val resolver = context.contentResolver
    return remember {
        Settings.Global.getFloat(
            resolver,
            Settings.Global.ANIMATOR_DURATION_SCALE,
            1f,
        ) == 0f
    }
}

// iosMain
@Composable
actual fun rememberReduceMotion(): Boolean {
    return remember { UIAccessibility.isReduceMotionEnabled }
}
```

### Conditional Animation

```kotlin
val reduceMotion = rememberReduceMotion()

AnimatedVisibility(
    visible = showContent,
    enter = if (reduceMotion) {
        EnterTransition.None
    } else {
        fadeIn(tween(AnimDuration.medium, easing = AnimEasing.standard)) +
            slideInVertically(animationSpec = tween(AnimDuration.medium, easing = AnimEasing.decelerate))
    },
) { content() }
```

- **Entry animations**: use `EnterTransition.None` / `ExitTransition.None` when reduce motion is enabled
- **State transitions**: use `snap()` instead of `tween()` / `spring()`
- **Shimmer**: replace with static placeholder (solid `surfaceContainerLow` background, no animation)
- **Material 3 built-in animations** (ripples, checkbox, switch) respect system settings automatically — no action needed

---

## Bottom Sheet & Dialog Patterns

### Modal Bottom Sheet

Use for selection, filters, and non-destructive choices:

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun <Feature>BottomSheet(
    onDismiss: () -> Unit,
    content: @Composable ColumnScope.() -> Unit,
) {
    val sheetState = rememberModalBottomSheetState()

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
        shape = RoundedCornerShape(topStart = Radius.xl, topEnd = Radius.xl),
    ) {
        Column(
            modifier = Modifier.padding(bottom = Spacing.xl),
            content = content,
        )
    }
}
```

### Confirmation Dialog

Use for destructive or irreversible actions:

```kotlin
@Composable
fun ConfirmationDialog(
    title: String,
    message: String,
    confirmLabel: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = { Text(message) },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text(
                    text = confirmLabel,
                    color = MaterialTheme.colorScheme.error,
                )
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text(stringResource(Res.string.core_action_cancel))
            }
        },
        modifier = modifier,
    )
}
```

**When to use which:**
- **Bottom sheets** — selection lists, filters, sort options, sharing, non-destructive multi-step flows
- **Dialogs** — confirmation of destructive actions (delete, discard), alerts, permission requests

---

## Gesture Convention Reference

Standard gesture mappings for CMP apps. Do not override platform conventions.

| Gesture | Expected Behavior | Notes |
|---------|-------------------|-------|
| Swipe from left edge | Navigate back | Platform back gesture — never override |
| Pull down on scrollable | Refresh content | Use `PullToRefreshBox` |
| Long press on item | Context menu / selection mode | Use haptic feedback |
| Pinch | Zoom (images/maps) | Only on zoomable content |
| Swipe on list item | Quick action (delete, archive) | Use `SwipeToDismissBox` |
| Tap | Primary action | Standard — no custom gesture needed |
| Double tap | Zoom in / like | Context-dependent, not required |

**Rules:**
- Never consume or override the system back gesture (swipe from left edge on iOS, predictive back on Android)
- Pull-to-refresh must use `PullToRefreshBox` — never custom fling detection
- Swipe-to-dismiss on list items should only dismiss toward end (right-to-left in LTR) to avoid conflict with back gesture
