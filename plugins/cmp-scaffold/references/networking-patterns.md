# Networking Patterns (Ktor 3.x)

Ktor 3.x HTTP client patterns for Kotlin Multiplatform. Replaces `Mock<Feature>RemoteDataSource` with a real network implementation.

---

## Version Catalog Entries

```toml
[versions]
ktor = "3.4.0"

[libraries]
ktor-client-core = { group = "io.ktor", name = "ktor-client-core", version.ref = "ktor" }
ktor-client-content-negotiation = { group = "io.ktor", name = "ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { group = "io.ktor", name = "ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging = { group = "io.ktor", name = "ktor-client-logging", version.ref = "ktor" }
ktor-client-okhttp = { group = "io.ktor", name = "ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { group = "io.ktor", name = "ktor-client-darwin", version.ref = "ktor" }
```

---

## HttpClient Factory

Place in `core/network/` module (shared across features):

```kotlin
package {package_base}.core.network

import io.ktor.client.HttpClient
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation
import io.ktor.client.plugins.logging.LogLevel
import io.ktor.client.plugins.logging.Logging
import io.ktor.client.plugins.HttpTimeout
import io.ktor.serialization.kotlinx.json.json
import kotlinx.serialization.json.Json

fun createHttpClient(): HttpClient = HttpClient {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
            prettyPrint = false
        })
    }
    install(Logging) {
        level = LogLevel.BODY
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
        connectTimeoutMillis = 15_000
    }
}
```

No `expect`/`actual` needed — Ktor engines are auto-discovered via platform dependencies.

---

## Safe API Call Utility

```kotlin
package {package_base}.core.network

import io.ktor.client.plugins.ClientRequestException
import io.ktor.client.plugins.ServerResponseException

suspend fun <T> safeApiCall(block: suspend () -> T): Result<T> = try {
    Result.success(block())
} catch (e: ClientRequestException) {
    Result.failure(e)
} catch (e: ServerResponseException) {
    Result.failure(e)
} catch (e: Exception) {
    Result.failure(e)
}
```

---

## Network DI Module

```kotlin
package {package_base}.core.network.di

import {package_base}.core.network.createHttpClient
import org.koin.dsl.module

val networkModule = module {
    single { createHttpClient() }
}
```

Add `networkModule` to AppModule `includes(...)`.

---

## DTO Pattern

DTOs live in `data/impl/datasource/remote/dto/` — separate from domain models in `data/api/model/`.

```kotlin
package {package_base}.<feature>.data.impl.datasource.remote.dto

import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName
import {package_base}.<feature>.data.api.model.<Model>

@Serializable
data class <Model>Dto(
    @SerialName("id") val id: String,
    // ... API fields
)

fun <Model>Dto.toDomain(): <Model> = <Model>(
    id = id,
    // ... field mapping
)
```

---

## Ktor Remote Data Source

Replaces `Mock<Feature>RemoteDataSource`:

```kotlin
package {package_base}.<feature>.data.impl.datasource.remote

import {package_base}.<feature>.data.api.model.<Model>
import {package_base}.<feature>.data.impl.datasource.remote.dto.<Model>Dto
import {package_base}.<feature>.data.impl.datasource.remote.dto.toDomain
import {package_base}.core.network.safeApiCall
import io.ktor.client.HttpClient
import io.ktor.client.call.body
import io.ktor.client.request.get

class Ktor<Feature>RemoteDataSource(
    private val httpClient: HttpClient
) : <Feature>RemoteDataSource {

    override suspend fun get<Feature>s(): Result<List<Model>> = safeApiCall {
        httpClient.get("https://api.example.com/<feature>s")
            .body<List<<Model>Dto>>()
            .map { it.toDomain() }
    }

    override suspend fun get<Feature>(id: String): Result<Model?> = safeApiCall {
        httpClient.get("https://api.example.com/<feature>s/$id")
            .body<<Model>Dto>()
            .toDomain()
    }
}
```

---

## DI Binding Update

Swap Mock for Ktor in the data module:

```kotlin
val <feature>DataModule = module {
    singleOf(::InMemory<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
    singleOf(::Ktor<Feature>RemoteDataSource) bind <Feature>RemoteDataSource::class
    singleOf(::<Feature>RepositoryImpl) bind <Feature>Repository::class
}
```

---

## build.gradle.kts Dependencies

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)
            implementation(libs.ktor.client.logging)
        }
        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
        }
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }
    }
}
```

---

## Upgrade Path: Mock → Ktor

1. Keep `Mock<Feature>RemoteDataSource` as test reference
2. Create `Ktor<Feature>RemoteDataSource` implementing the same `<Feature>RemoteDataSource` interface
3. Create DTO classes with `@Serializable` + `toDomain()` mappers
4. Swap DI binding: `Mock*` → `Ktor*`
5. Add `networkModule` to AppModule if not already present
6. The interface-based design means **zero changes to Repository, ViewModel, or Screen**
