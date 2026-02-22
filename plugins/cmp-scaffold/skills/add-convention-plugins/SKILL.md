---
name: add-convention-plugins
description: Set up a build-logic included build with convention plugins for consistent multi-module configuration
context: fork
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# Add Convention Plugins

Set up a `build-logic` included build with convention plugins to eliminate duplicated build configuration. See [convention-plugins-patterns.md](../../references/convention-plugins-patterns.md) for templates.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name`, existing module list. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}`. Read `gradle/libs.versions.toml` for current version references.

### 2. Extract Shared Config Values

Read 2-3 existing feature module `build.gradle.kts` files to identify:
- `compileSdk`, `minSdk`, `targetSdk`
- JVM target version
- Common plugins applied
- Shared dependencies (compose, KMP targets)
- Namespace pattern

### 3. Add Classpath Dependencies to Version Catalog

Add to `gradle/libs.versions.toml` if not present:
```toml
kotlin-gradle-plugin = { group = "org.jetbrains.kotlin", name = "kotlin-gradle-plugin", version.ref = "kotlin" }
compose-gradle-plugin = { group = "org.jetbrains.compose", name = "compose-gradle-plugin", version.ref = "compose-multiplatform" }
android-gradle-plugin = { group = "com.android.tools.build", name = "gradle", version.ref = "agp" }
```

### 4. Create build-logic Directory Structure

Create:
- `build-logic/settings.gradle.kts` — repos + version catalog import from parent
- `build-logic/gradle.properties`
- `build-logic/convention/build.gradle.kts` — `kotlin-dsl` plugin + classpath deps + gradlePlugin registration

### 5. Create Convention Plugins

Create in `build-logic/convention/src/main/kotlin/`:
- `AppConfig.kt` — shared constants (compileSdk, minSdk, targetSdk, jvmTarget)
- `KmpLibraryConventionPlugin.kt` — KMP + Android library config
- `ComposeFeatureConventionPlugin.kt` — KMP + Compose + Android library config
- `AndroidAppConventionPlugin.kt` — Android app module config

### 6. Update Root settings.gradle.kts

Add `includeBuild("build-logic")` to the `pluginManagement` block.

### 7. Migrate One Feature Module

Pick one feature module and update its `build.gradle.kts` to use the convention plugin as a proof-of-concept. Show the before/after diff.

### 8. Verify Build

Run `./gradlew :<migrated-module>:build` to verify the convention plugin works.

### 9. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `build-logic/settings.gradle.kts` imports parent version catalog
- [ ] `build-logic/convention/build.gradle.kts` registers all convention plugins in `gradlePlugin` block
- [ ] `AppConfig.kt` constants match values extracted from existing modules
- [ ] Root `settings.gradle.kts` has `includeBuild("build-logic")` in `pluginManagement`
- [ ] Migrated module builds successfully (`./gradlew :<module>:build`)

### 10. Report

Output a summary listing files created, the migrated module, and a migration guide for remaining modules. Include:
- Convention plugin IDs to use
- What config can be removed from each module's `build.gradle.kts`
- What config must remain (namespace, feature-specific deps)
