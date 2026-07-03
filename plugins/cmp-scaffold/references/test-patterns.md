# Test Patterns

Templates and conventions for unit test scaffolding in KMP feature modules.

---

## Test Directory Structure

```
<feature>/
├── data/impl/src/commonTest/kotlin/{package_base_path}/<feature>/data/impl/
│   ├── datasource/
│   │   ├── Fake<Feature>LocalDataSource.kt
│   │   └── Fake<Feature>RemoteDataSource.kt  (if remote exists)
│   └── repository/
│       └── <Feature>RepositoryImplTest.kt
├── domain/src/commonTest/kotlin/{package_base_path}/<feature>/domain/  (if exists)
│   └── usecases/
│       └── <UseCaseName>Test.kt
└── feature/src/commonTest/kotlin/{package_base_path}/<feature>/feature/
    ├── Fake<Feature>Repository.kt
    └── <Feature>ViewModelTest.kt
```

---

## Version Catalog Entries

```toml
[versions]
kotlinx-coroutines = "1.10.2"
turbine = "1.2.1"

[libraries]
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "kotlinx-coroutines" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
```

---

## Test Dependencies (build.gradle.kts)

Add to each module's `sourceSets` block:

```kotlin
commonTest.dependencies {
    implementation(kotlin("test"))
    implementation(libs.kotlinx.coroutines.test)
    implementation(libs.turbine)
}
```

Apply to:
- `<feature>/feature/build.gradle.kts` — ViewModel tests
- `<feature>/data/impl/build.gradle.kts` — Repository tests
- `<feature>/domain/build.gradle.kts` — UseCase tests (if domain exists)

---

## Fake Local Data Source

```kotlin
package {package_base}.<feature>.data.impl.datasource

import {package_base}.<feature>.data.impl.datasource.local.<Feature>LocalDataSource
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow

class Fake<Feature>LocalDataSource : <Feature>LocalDataSource {
    private val _data = MutableStateFlow<List<Model>>(emptyList())

    // Implement all interface methods delegating to _data
    override fun get<Feature>sFlow(): Flow<List<Model>> = _data
    override suspend fun add(item: Model) { _data.value = _data.value + item }
    override suspend fun remove(id: String) { _data.value = _data.value.filterNot { it.id == id } }
    override suspend fun insert<Feature>s(items: List<Model>) { _data.value = items }

    // Test helper methods
    fun emit(items: List<Model>) { _data.value = items }
    fun clear() { _data.value = emptyList() }
}
```

Mirror the `LocalDataSource` interface exactly. Add `emit()`, `clear()`, and `emitError()` helpers for test control.

If a `RemoteDataSource` exists, also create `Fake<Feature>RemoteDataSource` with `shouldFail` control.

---

## Fake Repository

```kotlin
package {package_base}.<feature>.feature

import {package_base}.<feature>.data.api.repository.<Feature>Repository
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow

class Fake<Feature>Repository : <Feature>Repository {
    private val _data = MutableStateFlow<List<Model>>(emptyList())

    override fun get<Feature>s(): Flow<List<Model>> = _data

    // Suspend methods with failure control
    var shouldFail = false
    var failureMessage = "Test error"

    override suspend fun add(item: Model): Result<Unit> =
        if (shouldFail) Result.failure(Exception(failureMessage))
        else { _data.value = _data.value + item; Result.success(Unit) }

    override suspend fun remove(id: String): Result<Unit> =
        if (shouldFail) Result.failure(Exception(failureMessage))
        else { _data.value = _data.value.filterNot { it.id == id }; Result.success(Unit) }

    // Test helpers
    fun emit(items: List<Model>) { _data.value = items }
}
```

---

## ViewModel Test

```kotlin
package {package_base}.<feature>.feature

import app.cash.turbine.test
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import kotlin.test.*
import {package_base}.core.feature.ui.UiMessage

@OptIn(ExperimentalCoroutinesApi::class)
class <Feature>ViewModelTest {
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var fakeRepository: Fake<Feature>Repository
    private lateinit var viewModel: <Feature>ViewModel

    @BeforeTest
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        fakeRepository = Fake<Feature>Repository()
        viewModel = <Feature>ViewModel(fakeRepository)
    }

    @AfterTest
    fun tearDown() { Dispatchers.resetMain() }

    @Test
    fun `initial state is Loading`() = runTest {
        viewModel.uiState.test {
            assertIs<<Feature>UiState.Loading>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `emits Success when repository has data`() = runTest {
        val testData = listOf(/* sample model instances */)
        fakeRepository.emit(testData)
        viewModel.uiState.test {
            awaitItem() // Skip Loading
            val state = awaitItem()
            assertIs<<Feature>UiState.Success>(state)
            assertEquals(testData.size, state.items.size)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `action sends error message on failure`() = runTest {
        // Substitute the ViewModel's actual failing action (e.g. `refresh()`, `add(item)`) —
        // `refresh()` here is illustrative, matching the "Local + Remote" refresh action
        // from data-patterns.md.
        fakeRepository.shouldFail = true
        viewModel.messages.test {
            viewModel.refresh()
            val message = awaitItem()
            assertEquals(UiMessage.Severity.ERROR, message.severity)
            cancelAndIgnoreRemainingEvents()
        }
    }

    // Generate tests for each ViewModel action method:
    // - Success case
    // - Failure case (verify `messages` Channel emission — assert severity, not exact text)
    // - State transitions
}
```

`viewModel.messages` is a `Flow<UiMessage>` backed by a `Channel` (see [code-templates.md](code-templates.md#uimessage-one-shot-severity-tagged-message)) — Turbine's `.test { }` collects it the same way as any other `Flow`. Assert on `message.severity` and, if needed, that `message.text` is the friendly copy passed as `userMessageFor`'s `default` — never assert against a raw exception message, since the mapper seam guarantees one is never surfaced.

> **`WhileSubscribed` timing caveat:** When the ViewModel uses `SharingStarted.WhileSubscribed(5_000)`, the `stateIn` flow only starts collecting when a subscriber is present. In tests, always await state transitions (e.g., `awaitItem()` for Loading, then `awaitItem()` for Success) **inside** the Turbine `test {}` block — not before it. Setting up fake data before calling `viewModel.uiState.test {}` ensures the data is ready when the subscription starts, but you must still await the Loading→Success transition within the block.

### Test Method Generation Guide

Generate a test for:
- Initial `Loading` state
- `Success` state with data
- `Empty` state (if UiState has an Empty variant)
- Each action method — success path
- Each action method — failure path (verify `messages` Channel emission via Turbine)

---

## Repository Test

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class <Feature>RepositoryImplTest {
    private lateinit var fakeLocalDataSource: Fake<Feature>LocalDataSource
    private lateinit var repository: <Feature>RepositoryImpl

    @BeforeTest
    fun setup() {
        fakeLocalDataSource = Fake<Feature>LocalDataSource()
        repository = <Feature>RepositoryImpl(fakeLocalDataSource)
    }

    @Test
    fun `getItems returns flow from local data source`() = runTest {
        val testData = listOf(/* ... */)
        fakeLocalDataSource.emit(testData)
        repository.get<Feature>s().test {
            assertEquals(testData, awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    // Test each repository method
}
```

If repository has a remote data source, also test:
- Cache-first: local data returned immediately
- Refresh updates local cache from remote
- Refresh failure handling

---

## UseCase Test (if domain layer exists)

```kotlin
class Get<Feature>sUseCaseTest {
    private lateinit var fakeRepository: Fake<Feature>Repository
    private lateinit var useCase: Get<Feature>sUseCase

    @BeforeTest
    fun setup() {
        fakeRepository = Fake<Feature>Repository()
        useCase = Get<Feature>sUseCase(fakeRepository)
    }

    @Test
    fun `invoke returns filtered data`() = runTest {
        // Test use case logic
    }
}
```
