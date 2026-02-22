# Image Loading Patterns (Coil 3)

Coil 3 image loading for Compose Multiplatform. Provides `AsyncImage` composable with cross-platform caching.

---

## Version Catalog Entries

```toml
[versions]
coil = "3.3.0"

[libraries]
coil-compose = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }
coil-network-ktor3 = { group = "io.coil-kt.coil3", name = "coil-network-ktor3", version.ref = "coil" }
```

---

## ImageLoader Setup

Configure once in the App composable:

```kotlin
package {package_base}

import coil3.ImageLoader
import coil3.compose.setSingletonImageLoaderFactory
import coil3.memory.MemoryCache
import coil3.request.crossfade

@Composable
fun App() {
    setSingletonImageLoaderFactory { context ->
        ImageLoader.Builder(context)
            .crossfade(true)
            .memoryCache {
                MemoryCache.Builder()
                    .maxSizePercent(context, 0.25)
                    .build()
            }
            .build()
    }
    AppTheme {
        AppNavHost()
    }
}
```

> **Note:** Coil 3 handles disk caching automatically per platform — no custom `platformCacheDirectory` expect/actual is needed. Omitting `.diskCache {}` uses Coil's built-in platform-appropriate cache location and a sensible default size. Only configure `.diskCache {}` if you need to override the cache path or size limit.

---

## Basic AsyncImage Usage

```kotlin
import coil3.compose.AsyncImage
import androidx.compose.foundation.layout.size

AsyncImage(
    model = item.imageUrl,
    contentDescription = stringResource(Res.string.<feature>_image_description),
    modifier = Modifier.size(IconSize.xl),
    contentScale = ContentScale.Crop,
)
```

---

## AsyncImage with Loading/Error States

```kotlin
import coil3.compose.SubcomposeAsyncImage
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme

SubcomposeAsyncImage(
    model = item.imageUrl,
    contentDescription = stringResource(Res.string.<feature>_image_description),
    modifier = Modifier.size(IconSize.xl),
    contentScale = ContentScale.Crop,
    loading = {
        CircularProgressIndicator(
            modifier = Modifier.size(IconSize.md),
            color = MaterialTheme.colorScheme.primary,
        )
    },
    error = {
        Icon(
            imageVector = Icons.Default.BrokenImage,
            contentDescription = null, // decorative
            modifier = Modifier.size(IconSize.lg),
            tint = MaterialTheme.colorScheme.onSurfaceVariant,
        )
    },
)
```

---

## Integration with Ktor

If the project already uses Ktor (see [networking-patterns.md](networking-patterns.md)), Coil can share the Ktor network layer via `coil-network-ktor3`:

```kotlin
// In ImageLoader setup
import coil3.network.ktor3.KtorNetworkFetcherFactory

ImageLoader.Builder(context)
    .components {
        add(KtorNetworkFetcherFactory(httpClient = get())) // from Koin
    }
    .build()
```

This ensures images use the same HttpClient (and its timeout/logging config).

---

## Where to Add Dependencies

### Feature module (if only one module needs images)

```kotlin
// <feature>/feature/build.gradle.kts
commonMain.dependencies {
    implementation(libs.coil.compose)
}
```

### Core module (if multiple features need images)

```kotlin
// core/feature/build.gradle.kts
commonMain.dependencies {
    api(libs.coil.compose)
    implementation(libs.coil.network.ktor3)
}
```

---

## Key Rules

- Always provide `contentDescription` (use `stringResource()`, never hardcoded) or explicit `null` for decorative images
- Use design tokens (`IconSize.*`, `Spacing.*`) for image sizing — no raw dp
- Use `ContentScale.Crop` for thumbnails/avatars, `ContentScale.Fit` for full images
- Set up `setSingletonImageLoaderFactory` exactly once in the root App composable
- Prefer `SubcomposeAsyncImage` when loading/error states are needed
