---
name: add-domain-layer
description: Add a domain/use-case layer to an existing feature module, separating business logic from ViewModel and data layers
argument-hint: <feature-name>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Domain Layer

Add a `domain` sub-module to an existing feature, extracting business logic into use case classes. Use cases sit between the ViewModel and Repository, encapsulating operations that involve multiple data sources, filtering, sorting, or business rules.

## Input

`$ARGUMENTS` — the feature name (e.g., `orders`, `products`, `cart`)

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Validate Existing Module

Verify the feature has a data layer (`<feature>/data/api/` with repository interface) and feature layer (`<feature>/feature/` with ViewModel). If missing, tell user to run `/cmp-scaffold:scaffold-feature <feature>` first.

### 3. Analyze the Module

Read the ViewModel to identify business logic that should be extracted:
- Data transformations (filtering, sorting, mapping) applied to repository flows
- Operations that combine data from multiple sources
- Complex action methods with multi-step logic
- Validation or business rules embedded in the ViewModel

Read the repository interface to understand available methods.

### 4. Create Domain Module Structure

Read [domain-layer-patterns.md](../../references/domain-layer-patterns.md) for the module structure template, `build.gradle.kts` template, and dependency rules.

Create `<feature>/domain/` following the structure from domain-layer-patterns.md. Key constraint: depends only on `:<feature>:data:api`, never on `:data:impl` or `:feature`.

### 5. Create Use Cases

Read [domain-layer-patterns.md](../../references/domain-layer-patterns.md) for use case templates (Flow-based observe pattern and suspend action pattern), naming conventions, and rules.

Create use cases based on the ViewModel analysis from Step 3. Each use case has a single `operator fun invoke(...)` method. Use `Flow<T>` for reactive streams, `Result<T>` for one-shot actions.

### 6. Create Domain DI Module

Read [domain-layer-patterns.md](../../references/domain-layer-patterns.md) for the DI module template. Use `factoryOf` for all use cases (stateless, new instance per injection).

### 7. Update ViewModel

Read [domain-layer-patterns.md](../../references/domain-layer-patterns.md) for the ViewModel with use cases template.

Replace direct repository usage with use case injection:
- Flow-based use cases are invoked directly (no `private val` needed — call in `stateIn` chain)
- Action use cases are stored as `private val` for repeated calls

### 8. Update Feature DI Module

Koin resolves use case dependencies automatically since they're registered in the domain module. No changes needed to `viewModelOf(::<Feature>ViewModel)` — Koin handles the constructor parameter resolution.

### 9. Update Feature build.gradle.kts

Add `project(":<feature>:domain")` to the feature module's dependencies. Remove `project(":<feature>:data:api")` from the feature module — the feature now depends on domain, which depends on data:api. Dependency chain: `feature → domain → data:api`.

### 10. Registration

- **settings.gradle.kts** — `include(":<feature>:domain")`
- **composeApp/build.gradle.kts** — `implementation(project(":<feature>:domain"))`
- **AppModule.kt** — add `<feature>DomainModule` to `includes(...)`

### 11. Verify

Before reporting, confirm each item — fix any violations:

- [ ] Domain module depends only on `:<feature>:data:api` — never on `:data:impl` or `:feature`
- [ ] Each use case has a single `operator fun invoke(...)` method
- [ ] Use cases registered with `factoryOf` in domain DI module (not `singleOf`)
- [ ] Feature module's `data:api` dependency removed (now transitive via domain)
- [ ] `:<feature>:domain` added to `settings.gradle.kts` and `composeApp/build.gradle.kts`
- [ ] Domain DI module included in `AppModule.kt`

### 12. Report

Output a summary listing files created, files modified, and use cases extracted. Suggest next steps:
- `/cmp-scaffold:scaffold-tests <feature>` to generate tests including use case tests
- Review the ViewModel — it should now be a thin coordinator, not contain business logic
