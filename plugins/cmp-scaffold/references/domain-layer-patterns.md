# Domain Layer Patterns

Use case patterns for Kotlin Multiplatform. Use cases sit between the ViewModel and Repository, encapsulating operations that involve multiple data sources, filtering, sorting, or business rules.

---

## Domain Module Structure

```
<feature>/domain/
└── src/commonMain/kotlin/{package_base_path}/<feature>/domain/
    ├── usecases/
    │   ├── Get<Feature>sUseCase.kt
    │   └── <Action><Feature>UseCase.kt
    └── di/
        └── <Feature>DomainModule.kt
```

### build.gradle.kts

```kotlin
plugins {
    id("{package_base}.kmp.library")
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(project(":<feature>:data:api"))
        }
    }
}
```

Dependencies:
- `project(":<feature>:data:api")` — accesses repository interface and models
- Must NOT depend on `:<feature>:data:impl` or `:<feature>:feature`

---

## Flow-Based Use Case (Observe)

For reactive data streams — returns `Flow<T>`:

```kotlin
class Get<Feature>sUseCase(
    private val repository: <Feature>Repository,
) {
    operator fun invoke(): Flow<List<Model>> =
        repository.observe<Feature>s()
            .map { items -> items.filter { /* business rule */ } }
}
```

Use when:
- Observing a data stream with transformations (filtering, sorting, mapping)
- Combining data from multiple repository flows
- Applying business rules to reactive data

---

## Suspend Use Case (Action)

For one-shot operations — returns `Result<T>`:

```kotlin
class Add<Feature>UseCase(
    private val repository: <Feature>Repository,
) {
    suspend operator fun invoke(item: Model): Result<Unit> =
        repository.add(item)
}
```

Use when:
- Creating, updating, or deleting data
- Operations that succeed or fail (wrapped in `Result`)
- Multi-step actions combining several repository calls

---

## Use Case Rules

- **One public method per use case** — `operator fun invoke(...)` only
- **Constructor injection** — takes repository interfaces, never concrete classes
- **Flow-based** use cases return `Flow<T>`, **action** use cases return `Result<T>` or `suspend` + `Result<T>`
- **Naming**: `<Verb><Feature>[s]UseCase` (e.g., `GetOrdersUseCase`, `PlaceOrderUseCase`, `ToggleFavoriteUseCase`)
- **Keep focused** — if a use case grows beyond ~20 lines, consider splitting
- **Stateless** — no mutable state; use `factoryOf` in DI (not `singleOf`)

---

## Domain DI Module

```kotlin
val <feature>DomainModule = module {
    factoryOf(::Get<Feature>sUseCase)
    factoryOf(::<Action><Feature>UseCase)
}
```

Use `factoryOf` for use cases (stateless, new instance per injection) rather than `singleOf`.

---

## ViewModel with Use Cases

Replace direct repository usage with use case injection:

```kotlin
class <Feature>ViewModel(
    get<Feature>s: Get<Feature>sUseCase,
    private val add<Feature>: Add<Feature>UseCase,
) : ViewModel() {
    val uiState: StateFlow<UiState> = get<Feature>s()
        .map { UiState.Success(it) as UiState }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
}
```

- Flow-based use cases are invoked directly (no `private val` needed — call in `stateIn` chain)
- Action use cases are stored as `private val` for repeated calls

---

## Dependency Chain

```
feature → domain → data:api
```

- Feature module depends on `domain`, which depends on `data:api`
- Feature no longer depends directly on `data:api` — domain is the intermediary
- `data:impl` is never depended on by domain or feature (only wired via DI)

---

## Registration

After creating the domain module:

1. **settings.gradle.kts** — `include(":<feature>:domain")`
2. **composeApp/build.gradle.kts** — `implementation(project(":<feature>:domain"))`
3. **AppModule.kt** — add `<feature>DomainModule` to `includes(...)`
4. **Feature build.gradle.kts** — add `project(":<feature>:domain")`, remove `project(":<feature>:data:api")`

---

## Key Rules

- Domain module has zero UI or framework dependencies — pure Kotlin + Coroutines + Koin
- Use cases are the only public API of the domain module
- Never put use cases in the data or feature layer — they belong exclusively in domain
- ViewModel becomes a thin coordinator: collects use case flows, dispatches use case actions
- Test use cases in isolation by mocking repository interfaces
