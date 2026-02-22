# UI Recipes: Loading, Empty, Error & State Transitions

Copy-paste-ready patterns for loading states, empty/error views, state transitions, and connectivity awareness. All commonMain-compatible, design-token-compliant.

---

## Shimmer Loading

Replace `CircularProgressIndicator` with shimmer placeholders that hint at content shape.

### ShimmerEffect Modifier

```kotlin
package {package_base}.core.feature.designsystem

import androidx.compose.animation.core.RepeatMode
import androidx.compose.animation.core.animateFloat
import androidx.compose.animation.core.infiniteRepeatable
import androidx.compose.animation.core.rememberInfiniteTransition
import androidx.compose.animation.core.tween
import androidx.compose.foundation.background
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.composed
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Brush

private const val ShimmerTranslateRange = 300f

fun Modifier.shimmer(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateX by transition.animateFloat(
        initialValue = -ShimmerTranslateRange,
        targetValue = ShimmerTranslateRange,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = AnimDuration.long),
            repeatMode = RepeatMode.Restart,
        ),
        label = "shimmerTranslate",
    )
    val shimmerColors = listOf(
        MaterialTheme.colorScheme.surfaceContainerLow,
        MaterialTheme.colorScheme.surfaceContainerHigh,
        MaterialTheme.colorScheme.surfaceContainerLow,
    )
    background(
        brush = Brush.linearGradient(
            colors = shimmerColors,
            start = Offset(translateX, 0f),
            end = Offset(translateX + ShimmerTranslateRange, 0f),
        ),
    )
}
```

### Shimmer Placeholder Item

```kotlin
/** Width fractions for shimmer placeholder lines. */
private object ShimmerPlaceholderFraction {
    const val title = 0.7f
    const val subtitle = 0.5f
}

@Composable
fun ShimmerItem(modifier: Modifier = Modifier) {
    Row(
        modifier = modifier
            .fillMaxWidth()
            .padding(Spacing.lg),
        horizontalArrangement = Arrangement.spacedBy(Spacing.md),
    ) {
        Box(
            modifier = Modifier
                .size(IconSize.xxxl)
                .clip(RoundedCornerShape(Radius.sm))
                .shimmer(),
        )
        Column(
            verticalArrangement = Arrangement.spacedBy(Spacing.sm),
            modifier = Modifier.weight(1f),
        ) {
            Box(
                modifier = Modifier
                    .fillMaxWidth(ShimmerPlaceholderFraction.title)
                    .height(Spacing.lg)
                    .clip(RoundedCornerShape(Radius.xs))
                    .shimmer(),
            )
            Box(
                modifier = Modifier
                    .fillMaxWidth(ShimmerPlaceholderFraction.subtitle)
                    .height(Spacing.md)
                    .clip(RoundedCornerShape(Radius.xs))
                    .shimmer(),
            )
        }
    }
}
```

### List Loading Placeholder

```kotlin
@Composable
fun ListLoadingPlaceholder(itemCount: Int = 6) {
    LazyColumn(userScrollEnabled = false) {
        items(itemCount) { ShimmerItem() }
    }
}
```

Adapt `ShimmerItem` shape to match actual content — grid items use square placeholders, cards use card-shaped placeholders, detail screens use header + body block placeholders.

---

## Empty States

Replace `Text("No items")` with a structured empty state.

```kotlin
package {package_base}.core.feature.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.style.TextAlign
import {package_base}.core.feature.designsystem.ContentAlpha
import {package_base}.core.feature.designsystem.IconSize
import {package_base}.core.feature.designsystem.Spacing

@Composable
fun EmptyStateView(
    icon: ImageVector,
    title: String,
    subtitle: String,
    modifier: Modifier = Modifier,
    actionLabel: String? = null,
    onAction: (() -> Unit)? = null,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(Spacing.xxl),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        Icon(
            imageVector = icon,
            contentDescription = null,
            modifier = Modifier.size(IconSize.xxl),
            tint = MaterialTheme.colorScheme.onSurfaceVariant
                .copy(alpha = ContentAlpha.medium),
        )
        Spacer(modifier = Modifier.height(Spacing.lg))
        Text(
            text = title,
            style = MaterialTheme.typography.titleMedium,
            textAlign = TextAlign.Center,
        )
        Spacer(modifier = Modifier.height(Spacing.sm))
        Text(
            text = subtitle,
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
                .copy(alpha = ContentAlpha.medium),
            textAlign = TextAlign.Center,
        )
        if (actionLabel != null && onAction != null) {
            Spacer(modifier = Modifier.height(Spacing.xl))
            TextButton(onClick = onAction) {
                Text(actionLabel)
            }
        }
    }
}
```

---

## Error States

Replace bare `Text(error)` with a recoverable error view.

```kotlin
package {package_base}.core.feature.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.outlined.ErrorOutline
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.style.TextAlign
import {package_base}.core.feature.designsystem.ContentAlpha
import {package_base}.core.feature.designsystem.IconSize
import {package_base}.core.feature.designsystem.Spacing

@Composable
fun ErrorStateView(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(Spacing.xxl),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        Icon(
            imageVector = Icons.Outlined.ErrorOutline,
            contentDescription = null,
            modifier = Modifier.size(IconSize.xxl),
            tint = MaterialTheme.colorScheme.error
                .copy(alpha = ContentAlpha.medium),
        )
        Spacer(modifier = Modifier.height(Spacing.lg))
        Text(
            text = message,
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant,
            textAlign = TextAlign.Center,
        )
        Spacer(modifier = Modifier.height(Spacing.xl))
        OutlinedButton(onClick = onRetry) {
            Text(stringResource(Res.string.core_action_retry))
        }
    }
}
```

---

## State Transitions

### AnimatedContent for UiState

```kotlin
AnimatedContent(
    targetState = uiState,
    transitionSpec = {
        fadeIn(tween(AnimDuration.medium)) togetherWith
            fadeOut(tween(AnimDuration.medium))
    },
    contentKey = { it::class }, // Only animate on structural state transitions, not data changes
    label = "<feature>StateTransition",
) { state ->
    when (state) {
        is <Feature>UiState.Loading -> ListLoadingPlaceholder()
        is <Feature>UiState.Success -> <Feature>Content(state.data)
        is <Feature>UiState.Error -> ErrorStateView(
            message = state.message,
            onRetry = onRetry,
        )
    }
}
```

### AnimatedVisibility (Conditional Elements)

```kotlin
AnimatedVisibility(
    visible = showFilter,
    enter = expandVertically(tween(AnimDuration.medium)) + fadeIn(),
    exit = shrinkVertically(tween(AnimDuration.medium)) + fadeOut(),
) {
    FilterChipRow(...)
}
```

### Crossfade (Simple Content Switch)

```kotlin
Crossfade(
    targetState = selectedTab,
    animationSpec = tween(AnimDuration.medium),
    label = "tabCrossfade",
) { tab ->
    when (tab) { ... }
}
```

Use `AnimatedContent` for UiState switching (most common). Use `Crossfade` for simple tab/content toggles. Use `AnimatedVisibility` for showing/hiding individual elements.

**Critical:** When `AnimatedContent` uses a sealed UiState as `targetState`, always include `contentKey = { it::class }`. Without it, `AnimatedContent` uses `.equals()` to detect changes — any data update within the same state type (e.g., toggling a favorite inside `Success`) triggers the fade transition, causing visible flicker. `contentKey` restricts animations to structural state transitions (Loading -> Success, Success -> Error).

---

## Connectivity Indicator

Show offline state and stale data non-intrusively. Integrates with cache-first architecture from [data-patterns.md](data-patterns.md).

### ConnectivityBanner

```kotlin
@Composable
fun ConnectivityBanner(
    isOffline: Boolean,
    lastUpdated: String?,
    modifier: Modifier = Modifier,
) {
    AnimatedVisibility(
        visible = isOffline,
        enter = expandVertically(tween(AnimDuration.medium, easing = AnimEasing.decelerate)) + fadeIn(),
        exit = shrinkVertically(tween(AnimDuration.medium, easing = AnimEasing.accelerate)) + fadeOut(),
        modifier = modifier,
    ) {
        Surface(
            color = MaterialTheme.colorScheme.errorContainer,
            modifier = Modifier.fillMaxWidth(),
        ) {
            Row(
                modifier = Modifier.padding(horizontal = Spacing.lg, vertical = Spacing.sm),
                horizontalArrangement = Arrangement.spacedBy(Spacing.sm),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                Icon(
                    imageVector = Icons.Outlined.CloudOff,
                    contentDescription = null,
                    modifier = Modifier.size(IconSize.sm),
                    tint = MaterialTheme.colorScheme.onErrorContainer,
                )
                Text(
                    text = if (lastUpdated != null) {
                        stringResource(Res.string.core_connectivity_offline_stale, lastUpdated)
                    } else {
                        stringResource(Res.string.core_connectivity_offline)
                    },
                    style = MaterialTheme.typography.labelMedium,
                    color = MaterialTheme.colorScheme.onErrorContainer,
                )
            }
        }
    }
}
```

Place `ConnectivityBanner` below the `TopAppBar` inside `Scaffold` content. The ViewModel exposes `isOffline` and `lastUpdated` from the repository's connectivity-aware state.
