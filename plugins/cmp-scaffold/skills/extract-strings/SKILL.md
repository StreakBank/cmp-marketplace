---
name: extract-strings
description: Extract hardcoded strings from composables into string resources with proper naming conventions
argument-hint: <feature-name-or-file-path>
context: fork
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Extract Strings to Resources

Scan composable files for hardcoded user-facing strings and extract them into `strings.xml`. See [string-resources.md](../../references/string-resources.md) for the naming convention and format reference.

## Input

`$ARGUMENTS` — feature name (e.g., `cart`, `products`) or a specific file path. If file, process that file only. If module name, process all composables.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`).

### 2. Find Target Files

If module name: glob for `<module>/feature/**/*Screen.kt`, `*View.kt`, `*Card.kt`, `*Bar.kt`, and `<module>/common/ui/**/*.kt`.

### 3. Read Existing strings.xml

Read `<module>/feature/src/commonMain/composeResources/values/strings.xml` for existing keys and patterns. Create if missing.

### 4. Detect Hardcoded Strings

**Extract:** `Text("...")`, `title = "..."`, `label = "..."`, `placeholder = { Text("...") }`, `contentDescription = "..."`, `message = "..."`, `subtitle = "..."`, `actionLabel = "..."`

**Skip:** Log/exception messages, programmatic interpolation, single formatting chars (`"$"`, `"%"`), package names, route strings, tag strings.

### 5. Generate Keys

Convention: `<module>_<screen>_<purpose>` (e.g., `cart_landing_title`, `cart_empty_title`, `cart_checkout_button`). See [string-resources.md](../../references/string-resources.md#naming-convention).

### 6. Update strings.xml

Group by screen with XML comments. For format arguments use `%1$s`, `%1$d`, `%1$.2f`.

### 7. Replace in Code

Replace with `stringResource(Res.string.<key>)`. For format args: `stringResource(Res.string.<key>, arg1, arg2)`.

### 8. Add Imports

```kotlin
import org.jetbrains.compose.resources.stringResource
import {resource_prefix}.<module>.feature.generated.resources.Res
import {resource_prefix}.<module>.feature.generated.resources.*
```

### 9. Report

Output: files processed with count per file, table of keys added (key, value, source file), totals (scanned, found, extracted, skipped), and any strings deliberately left as-is with reason.

Suggest next steps:
- `/cmp-scaffold:fix-imports` to verify resource import prefixes are correct
- `/cmp-quality:review-changes` to validate changes

## Troubleshooting

- `strings.xml` doesn't exist → create it with standard XML header and `<resources>` root element
- Malformed existing `strings.xml` → report the issue to the user, don't attempt auto-fix
- `Res.string` import not resolving → verify `compose.components.resources` dependency exists and resource prefix matches `rootProject.name`
