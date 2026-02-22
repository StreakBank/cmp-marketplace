---
name: upgrade-dependencies
description: Check and update dependency versions in the Gradle version catalog
context: fork
allowed-tools:
  - Read
  - Edit
  - Glob
  - Grep
  - Bash
---

# Upgrade Dependencies

Check for outdated dependencies in the Gradle version catalog and update them.

## Instructions

### 1. Read Version Catalog

Read `gradle/libs.versions.toml` to identify all current dependency versions.

### 2. Check for Updates

For each dependency group, check for the latest stable version:
- Try `./gradlew dependencyUpdates` if the plugin is available
- Otherwise, check key dependencies manually via web search

Focus on these critical KMP dependencies:
- Kotlin
- Compose Multiplatform
- AndroidX libraries (Lifecycle, Navigation, Room)
- Koin
- Ktor
- Coil
- KotlinX (Coroutines, Serialization, DateTime)
- AGP (Android Gradle Plugin)
- KSP

### 3. Categorize Updates

Group updates by risk level:
- **Patch** (e.g., 1.0.0 → 1.0.1) — safe, bug fixes only
- **Minor** (e.g., 1.0.0 → 1.1.0) — usually safe, may add APIs
- **Major** (e.g., 1.0.0 → 2.0.0) — breaking changes likely

### 4. Check Compatibility

For major updates, check for:
- Kotlin version requirements (many KMP libs require specific Kotlin versions)
- Compose Multiplatform ↔ Kotlin version compatibility
- KSP ↔ Kotlin version alignment (KSP versions include Kotlin version)

### 5. Update Version Catalog

Edit `gradle/libs.versions.toml` with the new versions. Start with patch/minor updates, then major updates.

### 6. Verify Build

Run `./gradlew build` to verify all updates work together. If the build fails:
- Identify which update caused the failure
- Check migration guides for breaking changes
- Fix or revert as needed

### 7. Report

Output a summary with:
- Table of updated dependencies (old version → new version, risk level)
- Any breaking changes encountered and how they were resolved
- Dependencies that were NOT updated (with reason, e.g., requires Kotlin X.Y)
- Build verification status

Suggest next steps:
- Run `dependency-audit` agent to verify module wiring after version changes
- Run `./gradlew build` to verify full build passes with updated dependencies
