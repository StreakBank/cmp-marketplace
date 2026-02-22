---
name: fix-imports
description: Fix CMP resource import prefixes across the project. Use after scaffolding or when resource imports are broken.
context: fork
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
---

# Fix CMP Resource Imports

Scan for incorrect Compose Multiplatform resource import prefixes and fix them.

## Background

CMP generates resource classes with a package prefix derived from `rootProject.name` in `settings.gradle.kts` (lowercased). Common mistake: using the directory name or hyphenated variant.

## Instructions

### 1. Discover Configuration

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Parse `include(...)` lines to find all modules with `composeResources/`.

Build mapping:
- Module `<name>` with `<name>/feature/src/commonMain/composeResources/` → correct prefix is `{resource_prefix}.<name>.feature.generated.resources`
- `composeApp/` → `{resource_prefix}.composeapp.generated.resources`

### 2. Find Wrong Prefixes

Search for imports containing `.generated.resources.` that don't start with `{resource_prefix}.`:
```
import (?!{resource_prefix}\.).*\.generated\.resources\.
```

Also check for: directory-name-based prefixes (underscores/hyphens where `rootProject.name` would be lowercased without them), double-dot typos (`{resource_prefix}..`).

### 3. Fix Each File

For each affected file, determine correct prefix from the mapping (based on file path → module). Replace wrong prefix with correct one using `replace_all`.

### 4. Validate References

Check that each `stringResource(Res.string.xxx)` references a key that exists in the module's `composeResources/values/strings.xml`. Report missing keys as warnings.

### 5. Report

Output: configuration (rootProject.name, prefix, modules discovered), results (files scanned, fixed, imports corrected), list of fixed files with counts, and any warnings about missing string keys.

Suggest next steps:
- `/cmp-scaffold:extract-strings` to extract any remaining hardcoded strings
- `/cmp-quality:review-changes` to validate changes

## Troubleshooting

- Generated resources not available → tell user to build first: `./gradlew :<module>:generateComposeResClass`
- Module not found in `settings.gradle.kts` → ask user for the correct module path
- Ambiguous resource prefix → read `rootProject.name` from `settings.gradle.kts` as the canonical source
