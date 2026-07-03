# Data Source Patterns

This project supports two data layer patterns. Choose the right one for each feature module.

---

## Pattern 1: Local-Only

Use when data lives entirely on-device (user preferences, local lists, draft content).

### Structure

```
<feature>/data/
├── api/
│   └── repository/<Feature>Repository.kt       # Interface (Flow reads, Result writes)
│   └── model/<Model>.kt                         # Data class
└── impl/
    ├── datasource/local/<Feature>LocalDataSource.kt  # Interface + InMemory impl
    ├── repository/<Feature>RepositoryImpl.kt         # Delegates to local DS
    └── di/<Feature>DataModule.kt                     # Koin bindings
```

### Data Flow

```
Screen → ViewModel → Repository → LocalDataSource
                          ↑                ↑
                      interface        interface
```

### DI Bindings

```kotlin
val <feature>DataModule = module {
    singleOf(::InMemory<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
    singleOf(::<Feature>RepositoryImpl) bind <Feature>Repository::class
}
```

### Key Points

- Repository reads return `Flow<T>` (reactive, from `MutableStateFlow`)
- Repository writes return `Result<Unit>` (one-shot, can fail)
- `InMemory*` prefix — swap to `Room*` / `DataStore*` later without changing interfaces
- All generated feature modules start with this pattern

---

## Pattern 2: Local + Remote (Cache-First)

Use when data comes from a backend API but should be available offline and load instantly.

### Structure

Adds to local-only:
```
<feature>/data/impl/
└── datasource/remote/<Feature>RemoteDataSource.kt  # Interface + Mock impl
```

### Data Flow

```
Screen → ViewModel → Repository ─read──→ LocalDataSource (cache)
                         │
                         └─refresh──→ RemoteDataSource → LocalDataSource
```

### Strategy: Cache-First

1. **Reads always come from local** — instant, offline-capable
2. **Refresh fetches from remote** and updates local cache
3. Observers of the local `Flow` automatically see updated data

### Repository Interface Addition

```kotlin
suspend fun refresh<Feature>s(): Result<Unit>
```

### DI Bindings

```kotlin
val <feature>DataModule = module {
    singleOf(::InMemory<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
    singleOf(::Mock<Feature>RemoteDataSource) bind <Feature>RemoteDataSource::class
    singleOf(::<Feature>RepositoryImpl) bind <Feature>Repository::class
}
```

### Remote Data Source Rules

- Interface + mock implementation in the **same file** (matches local pattern)
- All remote operations return `Result<T>` to represent network failures
- Mock implementation includes `simulateNetworkDelay()` for realistic testing
- Replace `Mock*` with `Ktor*` for production

### ViewModel Refresh (Optional)

```kotlin
init { refresh() }

fun refresh() {
    viewModelScope.launch {
        repository.refresh<Feature>s()
            .onFailure { _messages.trySend(UiMessage.error(userMessageFor(it, "Couldn't refresh. Try again."))) }
    }
}
```

`_messages` / `UiMessage` / `userMessageFor` are the `Channel<UiMessage>` + message-mapper seam from [code-templates.md](code-templates.md#uimessage-one-shot-severity-tagged-message) — never re-emit `it.message` directly.

### Offline UX Guidance

The cache-first pattern naturally supports offline use — reads always come from local storage. Enhance the UX by surfacing connectivity state:

**Repository additions:**

```kotlin
interface <Feature>Repository {
    fun observe<Feature>s(): Flow<List<<Model>>>
    suspend fun refresh<Feature>s(): Result<Unit>
    val isRefreshing: StateFlow<Boolean>
    val lastRefreshed: StateFlow<Instant?>
}
```

**ViewModel surfaces connectivity state:**

```kotlin
data class Success(
    val items: List<<Model>>,
    val isOffline: Boolean = false,
    val isRefreshing: Boolean = false,
    val lastUpdated: String? = null,
) : <Feature>UiState
```

**UI integration:** Use the `ConnectivityBanner` from [ui-recipes.md](ui-recipes.md) below the TopAppBar. Always show cached data immediately — indicate staleness non-intrusively with a timestamp, never block content behind a loading spinner when cached data is available.

---

## Upgrading Local-Only → Local+Remote

To upgrade, create the remote data source and update the repository. See [code-templates.md](code-templates.md#remote-data-source-interface--mock-impl-in-same-file) for the remote data source template and [code-templates.md](code-templates.md#repository-implementation-local--remote-cache-first) for the cache-first repository template.

---

## Pattern 3: Room Persistence

Use when data needs to survive app restarts (replacing `InMemory*` with `Room*`). Room implementation uses the same `<Feature>LocalDataSource` interface — zero changes to Repository, ViewModel, or Screen.

See [persistence-patterns.md](persistence-patterns.md) for full templates including Entity, DAO, Database, `@ConstructedBy`, platform builders, `BundledSQLiteDriver`, and DI bindings.

---

## Pattern 4: Ktor Networking

Use when replacing `Mock*` with real HTTP calls (replacing `Mock*` with `Ktor*`). Ktor implementation uses the same `<Feature>RemoteDataSource` interface — zero changes to Repository, ViewModel, or Screen.

See [networking-patterns.md](networking-patterns.md) for full templates including HttpClient factory, `safeApiCall`, DTOs, remote data source implementation, and DI bindings.

---

## Image Data in Models

Image URLs are stored as `String` fields in domain models (e.g., `imageUrl`, `thumbnailUrl`, `avatarUrl`). Key rules:

- **Domain model** — image URLs are plain `String` fields, no special types
- **UI layer** — handles loading and caching via Coil `AsyncImage` (see [image-loading-patterns.md](image-loading-patterns.md))
- **Room persistence** — store the URL string only; don't cache image bytes in the database. Coil manages its own disk cache.
- **DTOs** — image URL field names may differ from the domain model; map in `toDomain()`

---

## When to Use Which

| Scenario | Pattern |
|----------|---------|
| User's local list (cart, favorites, drafts) | Local-only |
| Backend-driven catalog (products, categories) | Local + Remote |
| User settings / preferences | Local-only |
| Content feed, search results | Local + Remote |
| Offline-first with sync | Local + Remote |
| Data survives app restarts | Local-only + Room (Pattern 3) |
| Real API integration | Local + Remote + Ktor (Pattern 4) |

---

## Upgrade Paths

```
InMemory* ──────→ Room*          (Pattern 1 → 3)
Mock*    ──────→ Ktor*           (Pattern 2 → 4)
InMemory + Mock → Room + Ktor   (Pattern 2 → 3 + 4)
```

All upgrades are interface-based — swap the implementation, update the DI binding, zero changes to consumers.
