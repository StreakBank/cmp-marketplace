# Code Templates

All code templates use `{package_base}`, `{resource_prefix}`, `<feature>`, and `<Feature>` (PascalCase) placeholders. Skills derive these values from `settings.gradle.kts` and `composeApp/build.gradle.kts` at runtime.

---

## Directory Structure

```
<feature>/
├── data/
│   ├── api/
│   │   ├── build.gradle.kts
│   │   └── src/commonMain/kotlin/{package_base_path}/<feature>/data/api/
│   │       ├── model/
│   │       └── repository/
│   └── impl/
│       ├── build.gradle.kts
│       └── src/commonMain/kotlin/{package_base_path}/<feature>/data/impl/
│           ├── datasource/local/
│           ├── repository/
│           └── di/
└── feature/
    ├── build.gradle.kts
    └── src/commonMain/kotlin/{package_base_path}/<feature>/feature/
        ├── <Feature>ViewModel.kt
        ├── <Feature>UiState.kt
        ├── <Feature>Screen.kt
        ├── navigation/
        │   └── <Feature>Navigation.kt
        └── di/
            └── <Feature>FeatureModule.kt
    └── src/commonMain/composeResources/values/
        └── strings.xml
```

---

## Model

```kotlin
package {package_base}.<feature>.data.api.model

data class <Model>(
    val id: String,
    // ... fields
)
```

---

## Repository Interface

```kotlin
package {package_base}.<feature>.data.api.repository

import {package_base}.<feature>.data.api.model.<Model>
import kotlinx.coroutines.flow.Flow

interface <Feature>Repository {
    fun get<Feature>s(): Flow<List<Model>>
    suspend fun add(item: <Model>): Result<Unit>
    suspend fun remove(id: String): Result<Unit>
}
```

---

## Local Data Source (Interface + InMemory impl in same file)

```kotlin
package {package_base}.<feature>.data.impl.datasource.local

import {package_base}.<feature>.data.api.model.<Model>
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow

interface <Feature>LocalDataSource {
    fun get<Feature>sFlow(): Flow<List<Model>>
    suspend fun add(item: <Model>)
    suspend fun remove(id: String)
    suspend fun insert<Feature>s(items: List<Model>)
}

class InMemory<Feature>LocalDataSource : <Feature>LocalDataSource {
    private val _data = MutableStateFlow<List<Model>>(emptyList())

    override fun get<Feature>sFlow(): Flow<List<Model>> = _data.asStateFlow()

    override suspend fun add(item: <Model>) {
        _data.value = _data.value + item
    }

    override suspend fun remove(id: String) {
        _data.value = _data.value.filterNot { it.id == id }
    }

    override suspend fun insert<Feature>s(items: List<Model>) {
        _data.value = items
    }
}
```

---

## Remote Data Source (Interface + Mock impl in same file)

```kotlin
package {package_base}.<feature>.data.impl.datasource.remote

import {package_base}.<feature>.data.api.model.<Model>
import kotlinx.coroutines.delay

interface <Feature>RemoteDataSource {
    suspend fun get<Feature>s(): Result<List<Model>>
    suspend fun get<Feature>(id: String): Result<Model?>
}

class Mock<Feature>RemoteDataSource : <Feature>RemoteDataSource {
    private suspend fun simulateNetworkDelay() { delay(500) }

    private val mockData = listOf<Model>(/* sample data */)

    override suspend fun get<Feature>s(): Result<List<Model>> = try {
        simulateNetworkDelay()
        Result.success(mockData)
    } catch (e: Exception) { Result.failure(e) }

    override suspend fun get<Feature>(id: String): Result<Model?> = try {
        simulateNetworkDelay()
        Result.success(mockData.find { it.id == id })
    } catch (e: Exception) { Result.failure(e) }
}
```

---

## Repository Implementation (Local-Only)

```kotlin
package {package_base}.<feature>.data.impl.repository

import {package_base}.<feature>.data.api.repository.<Feature>Repository
import {package_base}.<feature>.data.impl.datasource.local.<Feature>LocalDataSource

class <Feature>RepositoryImpl(
    private val localDataSource: <Feature>LocalDataSource
) : <Feature>Repository {
    override fun get<Feature>s() = localDataSource.get<Feature>sFlow()
    override suspend fun add(item: <Model>) = runCatching { localDataSource.add(item) }
    override suspend fun remove(id: String) = runCatching { localDataSource.remove(id) }
}
```

## Repository Implementation (Local + Remote, Cache-First)

```kotlin
class <Feature>RepositoryImpl(
    private val remoteDataSource: <Feature>RemoteDataSource,
    private val localDataSource: <Feature>LocalDataSource
) : <Feature>Repository {
    override fun get<Feature>s() = localDataSource.get<Feature>sFlow()

    override suspend fun refresh<Feature>s(): Result<Unit> = try {
        remoteDataSource.get<Feature>s().fold(
            onSuccess = { items -> localDataSource.insert<Feature>s(items); Result.success(Unit) },
            onFailure = { Result.failure(it) }
        )
    } catch (e: Exception) { Result.failure(e) }
}
```

---

## Data DI Module

```kotlin
package {package_base}.<feature>.data.impl.di

import org.koin.core.module.dsl.singleOf
import org.koin.dsl.bind
import org.koin.dsl.module

val <feature>DataModule = module {
    singleOf(::InMemory<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
    singleOf(::<Feature>RepositoryImpl) bind <Feature>Repository::class
}
```

---

## UiState

```kotlin
package {package_base}.<feature>.feature

sealed interface <Feature>UiState {
    data object Loading : <Feature>UiState
    data class Success(val data: List<Model>) : <Feature>UiState
    data class Error(val message: String) : <Feature>UiState
}
```

---

## ViewModel

```kotlin
package {package_base}.<feature>.feature

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class <Feature>ViewModel(
    private val repository: <Feature>Repository
) : ViewModel() {
    private val _errorEvents = MutableSharedFlow<String>()
    val errorEvents: SharedFlow<String> = _errorEvents.asSharedFlow()

    val uiState: StateFlow<<Feature>UiState> = repository.get<Feature>s()
        .map { data -> <Feature>UiState.Success(data) as <Feature>UiState }
        .catch { emit(<Feature>UiState.Error(it.message ?: "Unknown error")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), <Feature>UiState.Loading)
}
```

---

## Screen Composable (Key Requirements)

```kotlin
package {package_base}.<feature>.feature

// Key requirements:
// - koinViewModel() for ViewModel injection
// - collectAsStateWithLifecycle() for state
// - SnackbarHostState + LaunchedEffect for errorEvents
// - when expression handling all UiState variants
// - Spacing.* / IconSize.* for sizing (no raw dp)
// - stringResource(Res.string.xxx) for text (no hardcoded strings)
// - EmptyStateView / ErrorStateView from core/feature/ui/ if applicable
// - contentDescription on all Icon/Image/AsyncImage:
//     - Informational: contentDescription = stringResource(Res.string.xxx)
//     - Decorative: contentDescription = null
// - key parameter in LazyColumn/LazyVerticalGrid items() calls
```

---

## Navigation (Nav 2.x)

Requires the `kotlinx-serialization` plugin for `@Serializable` routes:
```toml
# libs.versions.toml
[plugins]
kotlinx-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

```kotlin
package {package_base}.<feature>.feature.navigation

import kotlinx.serialization.Serializable
import androidx.navigation.NavController
import androidx.navigation.NavGraphBuilder
import androidx.navigation.compose.composable
import androidx.navigation.compose.navigation

@Serializable data object <Feature>GraphRoute
@Serializable data object <Feature>Route

fun NavGraphBuilder.<feature>Graph(navController: NavController) {
    navigation<<Feature>GraphRoute>(startDestination = <Feature>Route::class) {
        composable<<Feature>Route> { <Feature>Screen() }
    }
}

fun NavController.navigateTo<Feature>() = navigate(<Feature>Route)
```

---

## Feature DI Module

```kotlin
package {package_base}.<feature>.feature.di

import org.koin.core.module.dsl.viewModelOf
import org.koin.dsl.module

val <feature>FeatureModule = module {
    viewModelOf(::<Feature>ViewModel)
}
```

---

## strings.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<resources>
    <!-- <Feature>Screen -->
    <string name="<feature>_title">...</string>
</resources>
```

Import pattern:
```kotlin
import {resource_prefix}.<feature>.feature.generated.resources.Res
import {resource_prefix}.<feature>.feature.generated.resources.*
```

---

## Preview

```kotlin
import org.jetbrains.compose.ui.tooling.preview.Preview

@Preview
@Composable
fun <Feature>ScreenPreview() {
    AppTheme {
        <Feature>ScreenContent(
            uiState = <Feature>UiState.Success(data = sampleData),
            onAction = {},
        )
    }
}
```

Place previews in the same file as the composable, after the main composable function. The unified `@Preview` annotation from `org.jetbrains.compose.ui.tooling.preview` works in commonMain across all targets.

Required dependency:
```kotlin
commonMain.dependencies {
    implementation(libs.compose.components.uiToolingPreview)
}
```

---

## build.gradle.kts Patterns

Feature module dependencies:
- `project(":core:feature")`
- `project(":<feature>:data:api")`
- `libs.compose.components.resources`

> **CMP 1.10+ note:** The `compose.xxx` shorthand accessors (e.g., `compose.runtime`, `compose.material3`) are deprecated. For new projects, prefer version catalog references (e.g., `libs.compose.runtime`). Existing projects can continue using the shorthand until migration to convention plugins.

Data API module: typically a pure Kotlin module with model classes.

Data impl module dependencies:
- `project(":<feature>:data:api")`

---

## Registration

**settings.gradle.kts:**
```kotlin
include(":<feature>:data:api")
include(":<feature>:data:impl")
include(":<feature>:feature")
```

**composeApp/build.gradle.kts:**
```kotlin
implementation(project(":<feature>:data:api"))
implementation(project(":<feature>:data:impl"))
implementation(project(":<feature>:feature"))
```

**AppModule.kt:**
```kotlin
includes(<feature>DataModule, <feature>FeatureModule)
```
