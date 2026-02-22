# UI Recipes: Lists & Search

Copy-paste-ready list patterns and search components. All commonMain-compatible, design-token-compliant.

---

## Staggered Entry Animation

```kotlin
@Composable
fun <T> AnimatedLazyColumn(
    items: List<T>,
    key: (T) -> Any,
    modifier: Modifier = Modifier,
    contentPadding: PaddingValues = PaddingValues(Spacing.lg),
    itemContent: @Composable (T) -> Unit,
) {
    LazyColumn(
        modifier = modifier,
        contentPadding = contentPadding,
        verticalArrangement = Arrangement.spacedBy(Spacing.sm),
    ) {
        itemsIndexed(items, key = { _, item -> key(item) }) { index, item ->
            val visibleState = remember {
                MutableTransitionState(false).apply { targetState = true }
            }
            AnimatedVisibility(
                visibleState = visibleState,
                enter = fadeIn(
                    animationSpec = tween(
                        durationMillis = AnimDuration.medium,
                        delayMillis = index * 50,
                    ),
                ) + slideInVertically(
                    initialOffsetY = { it / 4 },
                    animationSpec = tween(
                        durationMillis = AnimDuration.medium,
                        delayMillis = index * 50,
                    ),
                ),
            ) {
                itemContent(item)
            }
        }
    }
}
```

## Swipe-to-Dismiss

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun <T> SwipeToDismissItem(
    item: T,
    onDismiss: (T) -> Unit,
    content: @Composable (T) -> Unit,
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDismiss(item)
                true
            } else false
        },
    )
    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.errorContainer)
                    .padding(horizontal = Spacing.xl),
                contentAlignment = Alignment.CenterEnd,
            ) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = stringResource(Res.string.core_action_delete),
                    tint = MaterialTheme.colorScheme.onErrorContainer,
                )
            }
        },
        enableDismissFromStartToEnd = false,
    ) {
        content(item)
    }
}
```

## Sticky Section Headers

```kotlin
@Composable
fun <T> SectionedLazyColumn(
    sections: Map<String, List<T>>,
    key: (T) -> Any,
    modifier: Modifier = Modifier,
    itemContent: @Composable (T) -> Unit,
) {
    LazyColumn(
        modifier = modifier,
        contentPadding = PaddingValues(vertical = Spacing.sm),
    ) {
        sections.forEach { (header, sectionItems) ->
            stickyHeader(key = header) {
                Surface(
                    color = MaterialTheme.colorScheme.surface,
                    tonalElevation = Elevation.sm,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text(
                        text = header,
                        style = MaterialTheme.typography.labelLarge,
                        color = MaterialTheme.colorScheme.primary,
                        modifier = Modifier.padding(
                            horizontal = Spacing.lg,
                            vertical = Spacing.sm,
                        ),
                    )
                }
            }
            items(sectionItems, key = { key(it) }) { item ->
                itemContent(item)
            }
        }
    }
}
```

## Pull-to-Refresh

```kotlin
import androidx.compose.material3.pulltorefresh.PullToRefreshBox
import androidx.compose.material3.ExperimentalMaterial3Api

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableContent(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit,
) {
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh,
        modifier = modifier,
    ) {
        content()
    }
}
```

`PullToRefreshBox` is the CMP-compatible Material 3 pull-to-refresh. Wrap any scrollable content. The `isRefreshing` flag typically maps to a ViewModel state field.

---

## Animated SearchBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun <Feature>SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: (String) -> Unit,
    expanded: Boolean,
    onExpandedChange: (Boolean) -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit,
) {
    SearchBar(
        inputField = {
            SearchBarDefaults.InputField(
                query = query,
                onQueryChange = onQueryChange,
                onSearch = onSearch,
                expanded = expanded,
                onExpandedChange = onExpandedChange,
                placeholder = {
                    Text(stringResource(Res.string.<feature>_search_placeholder))
                },
                leadingIcon = {
                    Icon(Icons.Default.Search, contentDescription = null)
                },
                trailingIcon = {
                    if (query.isNotEmpty()) {
                        IconButton(onClick = { onQueryChange("") }) {
                            Icon(
                                Icons.Default.Clear,
                                contentDescription = stringResource(Res.string.core_action_clear),
                            )
                        }
                    }
                },
            )
        },
        expanded = expanded,
        onExpandedChange = onExpandedChange,
        modifier = modifier,
    ) {
        content()
    }
}
```

## Filter Chips Row

```kotlin
@Composable
fun FilterChipRow(
    filters: List<String>,
    selectedFilter: String?,
    onFilterSelected: (String?) -> Unit,
    modifier: Modifier = Modifier,
) {
    LazyRow(
        modifier = modifier,
        horizontalArrangement = Arrangement.spacedBy(Spacing.sm),
        contentPadding = PaddingValues(horizontal = Spacing.lg),
    ) {
        items(filters) { filter ->
            FilterChip(
                selected = filter == selectedFilter,
                onClick = {
                    onFilterSelected(if (filter == selectedFilter) null else filter)
                },
                label = { Text(filter) },
                leadingIcon = if (filter == selectedFilter) {
                    {
                        Icon(
                            Icons.Default.Check,
                            contentDescription = null,
                            modifier = Modifier.size(IconSize.sm),
                        )
                    }
                } else null,
            )
        }
    }
}
```
