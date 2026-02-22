# Persistence Patterns (Room KMP)

Room KMP patterns for persistent local storage. Replaces `InMemory<Feature>LocalDataSource` with Room-backed persistence.

---

## Version Catalog Entries

```toml
[versions]
room = "2.8.3"
sqlite = "2.6.2"
ksp = "2.3.5"

[libraries]
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
sqlite-bundled = { group = "androidx.sqlite", name = "sqlite-bundled", version.ref = "sqlite" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
room = { id = "androidx.room", version.ref = "room" }
```

> **Note:** KSP 2.x uses standalone semantic versioning (e.g., `2.3.5`), decoupled from the Kotlin compiler version.

---

## Entity

Place in `data/impl/datasource/local/db/`:

```kotlin
package {package_base}.<feature>.data.impl.datasource.local.db

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "<feature>s")
data class <Feature>Entity(
    @PrimaryKey val id: String,
    // ... fields matching domain model
)
```

---

## Entity ↔ Domain Mappers

```kotlin
package {package_base}.<feature>.data.impl.datasource.local.db

import {package_base}.<feature>.data.api.model.<Model>

fun <Feature>Entity.toDomain(): <Model> = <Model>(
    id = id,
    // ... field mapping
)

fun <Model>.toEntity(): <Feature>Entity = <Feature>Entity(
    id = id,
    // ... field mapping
)
```

---

## DAO

```kotlin
package {package_base}.<feature>.data.impl.datasource.local.db

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface <Feature>Dao {
    @Query("SELECT * FROM <feature>s")
    fun getAll(): Flow<List<<Feature>Entity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(items: List<<Feature>Entity>)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(item: <Feature>Entity)

    @Query("DELETE FROM <feature>s WHERE id = :id")
    suspend fun deleteById(id: String)
}
```

---

## Database

Room KMP requires `@ConstructedBy` for non-Android platforms. KSP auto-generates the `actual object` implementation.

```kotlin
package {package_base}.<feature>.data.impl.datasource.local.db

import androidx.room.ConstructedBy
import androidx.room.Database
import androidx.room.RoomDatabase
import androidx.room.RoomDatabaseConstructor

@Database(entities = [<Feature>Entity::class], version = 1)
@ConstructedBy(<Feature>DatabaseConstructor::class)
abstract class <Feature>Database : RoomDatabase() {
    abstract fun <feature>Dao(): <Feature>Dao
}

expect object <Feature>DatabaseConstructor : RoomDatabaseConstructor<<Feature>Database>
```

> **Note:** The `expect object` declaration produces a Beta warning. This is expected and can be suppressed with `-Xexpect-actual-classes` if desired.

---

## Platform Database Module (Koin DI)

Android requires `Context` for Room's `databaseBuilder`. iOS/non-Android platforms require an explicit `BundledSQLiteDriver`. Use expect/actual Koin Module to handle platform-specific construction.

### expect (commonMain)

```kotlin
package {package_base}.<feature>.data.impl.di

import org.koin.core.module.Module

expect val <feature>DatabaseModule: Module
```

### actual (androidMain)

```kotlin
package {package_base}.<feature>.data.impl.di

import android.content.Context
import androidx.room.Room
import {package_base}.<feature>.data.impl.datasource.local.db.<Feature>Database
import org.koin.core.module.Module
import org.koin.dsl.module

actual val <feature>DatabaseModule: Module = module {
    single {
        Room.databaseBuilder(
            context = get<Context>(),
            klass = <Feature>Database::class.java,
            name = "<feature>-database"
        ).build()
    }
    single { get<<Feature>Database>().<feature>Dao() }
}
```

### actual (iosMain)

```kotlin
package {package_base}.<feature>.data.impl.di

import androidx.room.Room
import androidx.sqlite.driver.bundled.BundledSQLiteDriver
import {package_base}.<feature>.data.impl.datasource.local.db.<Feature>Database
import org.koin.core.module.Module
import org.koin.dsl.module
import platform.Foundation.NSHomeDirectory

actual val <feature>DatabaseModule: Module = module {
    single {
        Room.databaseBuilder<<Feature>Database>(
            name = NSHomeDirectory() + "/<feature>-database",
        )
            .setDriver(BundledSQLiteDriver())
            .build()
    }
    single { get<<Feature>Database>().<feature>Dao() }
}
```

---

## Koin Initialization (required for Android Context)

Room KMP on Android requires `androidContext()` in Koin, which means using `startKoin` in platform entry points instead of `KoinApplication` composable.

### Android — Application class

```kotlin
class ExampleApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@ExampleApplication)
            modules(appModule, <feature>DatabaseModule, /* ... */)
        }
    }
}
```

Register in `AndroidManifest.xml`: `<application android:name=".ExampleApplication" ...>`

### iOS — MainViewController

```kotlin
private var koinInitialized = false
private fun initKoin() {
    if (!koinInitialized) {
        startKoin { modules(appModule, <feature>DatabaseModule, /* ... */) }
        koinInitialized = true
    }
}

fun MainViewController(): UIViewController {
    initKoin()
    return ComposeUIViewController { ExampleApp() }
}
```

> **Note:** With `startKoin`, the Compose Koin context is auto-configured. Remove any `KoinContext` composable wrapper — it is deprecated in Koin 4.x.

---

## Room Local Data Source

Implements the existing `<Feature>LocalDataSource` interface:

```kotlin
package {package_base}.<feature>.data.impl.datasource.local

import {package_base}.<feature>.data.api.model.<Model>
import {package_base}.<feature>.data.impl.datasource.local.db.<Feature>Dao
import {package_base}.<feature>.data.impl.datasource.local.db.toDomain
import {package_base}.<feature>.data.impl.datasource.local.db.toEntity
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

class Room<Feature>LocalDataSource(
    private val dao: <Feature>Dao
) : <Feature>LocalDataSource {

    override fun get<Feature>sFlow(): Flow<List<Model>> =
        dao.getAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun add(item: <Model>) {
        dao.insert(item.toEntity())
    }

    override suspend fun remove(id: String) {
        dao.deleteById(id)
    }

    override suspend fun insert<Feature>s(items: List<Model>) {
        dao.insertAll(items.map { it.toEntity() })
    }
}
```

---

## DI Module Update

Update the existing data module to swap InMemory → Room:

```kotlin
val <feature>DataModule = module {
    // Data sources — swap InMemory* for Room*
    singleOf(::Room<Feature>LocalDataSource) bind <Feature>LocalDataSource::class
    singleOf(::<Feature>RepositoryImpl) bind <Feature>Repository::class
}
```

> **Important:** The database creation and DAO provision are in the separate `<feature>DatabaseModule` (see above). Both modules must be included in the Koin initialization.

---

## build.gradle.kts Configuration

```kotlin
plugins {
    alias(libs.plugins.ksp)
    alias(libs.plugins.room)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.room.runtime)
            implementation(libs.sqlite.bundled)
        }
    }
}

room {
    schemaDirectory("$projectDir/schemas")
}

dependencies {
    listOf("kspAndroid", "kspIosX64", "kspIosArm64", "kspIosSimulatorArm64").forEach {
        add(it, libs.room.compiler)
    }
}
```

---

## Upgrade Path: InMemory → Room

1. Keep `InMemory<Feature>LocalDataSource` for unit testing
2. Create Entity, DAO, Database in `data/impl/datasource/local/db/`
3. Create expect/actual `<feature>DatabaseModule` for platform-specific database construction
4. Create `Room<Feature>LocalDataSource` implementing the same `<Feature>LocalDataSource` interface
5. Swap DI binding in `<feature>DataModule`: `InMemory*` → `Room*`
6. Add `<feature>DatabaseModule` to platform `startKoin` initialization
7. The interface-based design means **zero changes to Repository, ViewModel, or Screen**

> **Pre-requisite:** Ensure `startKoin` is used in platform entry points (not `KoinApplication` composable) for `androidContext()` support. See "Koin Initialization" section above.
