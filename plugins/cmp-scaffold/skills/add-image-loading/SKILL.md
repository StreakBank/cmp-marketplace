---
name: add-image-loading
description: Add Coil 3 image loading to a feature module's screen composables
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Image Loading

Add Coil 3 image loading to a feature module. See [image-loading-patterns.md](../../references/image-loading-patterns.md) for templates and [design-tokens.md](../../references/design-tokens.md) for sizing tokens.

## Input

`$ARGUMENTS` — the feature name (e.g., `products`, `profile`, `catalog`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check for Existing Coil Setup

Glob for `setSingletonImageLoaderFactory` across the codebase. If not found, this is the first image loading setup.

### 3. First-Time Setup (if needed)

If Coil is not yet configured:
- Add Coil version catalog entries to `gradle/libs.versions.toml`
- Set up `setSingletonImageLoaderFactory` in the App composable (read the current App.kt first)
- Add `coil-compose` dependency to the appropriate module

> **Note:** Coil handles disk caching automatically — no custom `platformCacheDirectory` expect/actual is needed.

### 4. Add Feature Dependency

Add `libs.coil.compose` to the feature module's `build.gradle.kts` commonMain dependencies.

### 5. Identify Image Fields

Read model classes in `<feature>/data/api/model/` and look for URL fields: `imageUrl`, `thumbnailUrl`, `avatarUrl`, `photoUrl`, `iconUrl`, or similar. Report which fields were found.

### 6. Update Screen Composables

Read existing screen composables in `<feature>/feature/`. For each composable that displays items with image URLs:
- Add `AsyncImage` or `SubcomposeAsyncImage` (if loading/error states are needed)
- Use design tokens for sizing (`IconSize.*`, `Spacing.*`)
- Provide `contentDescription` via `stringResource()` or explicit `null` for decorative images
- Use `ContentScale.Crop` for thumbnails, `ContentScale.Fit` for full images

### 7. Add String Resources

If new `contentDescription` strings are needed, add them to the feature's `strings.xml`.

### 8. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `setSingletonImageLoaderFactory` configured in root App composable
- [ ] Image composables use design tokens for sizing (`IconSize.*`, `Spacing.*`) — no raw dp
- [ ] Every `AsyncImage`/`SubcomposeAsyncImage` has explicit `contentDescription` (or `null` for decorative)
- [ ] Coil dependency added via version catalog reference
- [ ] No custom `platformCacheDirectory` expect/actual (Coil handles caching automatically)

### 9. Report

Output a summary listing files created, files modified, image fields found, and next steps:
- `/cmp-scaffold:polish-ui <feature>` to upgrade cards with rich image compositions
- Configure Ktor integration for shared networking (see networking-patterns.md)
- Consider pre-fetching images for lists

## Troubleshooting

- `setSingletonImageLoaderFactory` already configured → skip global setup, only add `AsyncImage` usage to the feature
- No image URL fields found in model classes → ask user which field holds the image URL
- Coil dependencies already in version catalog → skip catalog entries, just add the module-level dependency
