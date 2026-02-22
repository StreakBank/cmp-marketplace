# String Resources

All user-facing text in composables must use `stringResource()`. No hardcoded strings.

## Per-Module Resource Location

```
<module>/feature/src/commonMain/composeResources/values/strings.xml
```

## strings.xml Format

```xml
<?xml version="1.0" encoding="UTF-8"?>
<resources>
    <!-- <ScreenName> -->
    <string name="<module>_<screen>_<purpose>">The actual string value</string>
</resources>
```

Group entries by screen with XML comments.

## Naming Convention

```
<module>_<screen>_<purpose>
```

| Part | Rule | Example |
|------|------|---------|
| `<module>` | Feature module name | `cart`, `products` |
| `<screen>` | Screen or component context | `landing`, `detail`, `empty` |
| `<purpose>` | What the string is for | `title`, `subtitle`, `button`, `description`, `placeholder`, `label` |

### Examples

| Context | Key |
|---------|-----|
| `Text("Shopping Cart")` in CartLandingScreen | `cart_landing_title` |
| `Text("No items in cart")` in EmptyCartView | `cart_empty_title` |
| `contentDescription = "Remove item"` | `cart_item_remove_description` |
| `Text("Checkout")` button | `cart_checkout_button` |
| `placeholder = { Text("Search...") }` | `products_search_placeholder` |

### Rules

- Always prefix with module name
- Include screen/component context
- Describe the purpose (title, subtitle, button, description, placeholder, label)
- Use `snake_case`
- Keep concise but descriptive

## Format Arguments

For dynamic values, use positional placeholders in XML:
- `%1$s` — string
- `%1$d` — integer
- `%1$.2f` — decimal

```xml
<string name="cart_items_count">%1$d items in cart</string>
<string name="cart_total">Total: $%1$.2f</string>
```

In code:
```kotlin
Text(stringResource(Res.string.cart_items_count, items.size))
Text(stringResource(Res.string.cart_total, totalPrice))
```

## Import Pattern

Use the `{resource_prefix}` (from `rootProject.name`, lowercased):

```kotlin
import org.jetbrains.compose.resources.stringResource
import {resource_prefix}.<module>.feature.generated.resources.Res
import {resource_prefix}.<module>.feature.generated.resources.*
```

**Common mistakes to avoid:**
- Using the directory name instead of `rootProject.name` (e.g., `my_app.` instead of `myapp.`)
- Using a hyphenated variant (e.g., `my-app.` instead of `myapp.`)

## What to Extract vs Skip

**Extract (user-facing):**
- `Text("...")`, `title = "..."`, `label = "..."`, `subtitle = "..."`
- `placeholder = { Text("...") }`, `contentDescription = "..."`
- `message = "..."` (Snackbar, dialogs), `actionLabel = "..."`

**Skip (not user-facing):**
- Log messages, exception messages (internal)
- String interpolation that's purely programmatic
- Single formatting characters (`"$"`, `"%"`, `"x"`)
- Package names, route strings, tag strings
