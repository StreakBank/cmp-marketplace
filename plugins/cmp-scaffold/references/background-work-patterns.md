# Background Work Patterns

Background sync and scheduled work patterns for Kotlin Multiplatform. Uses expect/actual to bridge Android WorkManager and iOS BGTaskScheduler.

---

## Version Catalog Entries

```toml
[versions]
work = "2.10.1"

[libraries]
work-runtime-ktx = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "work" }
```

> WorkManager is Android-only. iOS uses BGTaskScheduler (system framework, no library dependency).

---

## Retry with Exponential Backoff (commonMain)

Place in `core/sync/` module — shared across features:

```kotlin
package {package_base}.core.sync

import kotlinx.coroutines.delay

suspend fun <T> retryWithBackoff(
    maxRetries: Int = 3,
    initialDelayMs: Long = 1_000,
    maxDelayMs: Long = 30_000,
    factor: Double = 2.0,
    block: suspend () -> T,
): T {
    var currentDelay = initialDelayMs
    repeat(maxRetries - 1) {
        try { return block() }
        catch (e: Exception) {
            delay(currentDelay)
            currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelayMs)
        }
    }
    return block() // final attempt — let exception propagate
}
```

---

## SyncUseCase Interface + SyncRegistry (commonMain)

```kotlin
package {package_base}.core.sync

interface SyncUseCase { suspend fun sync() }

class SyncRegistry {
    private val useCases = mutableMapOf<String, SyncUseCase>()
    fun register(tag: String, useCase: SyncUseCase) { useCases[tag] = useCase }
    fun get(tag: String): SyncUseCase? = useCases[tag]
}
```

---

## SyncScheduler Interface (commonMain)

```kotlin
package {package_base}.core.sync

interface SyncScheduler {
    fun scheduleOneTime(tag: String)
    fun schedulePeriodic(tag: String, intervalMinutes: Long)
    fun cancel(tag: String)
}
```

---

## AndroidSyncScheduler (androidMain)

```kotlin
package {package_base}.core.sync

import android.content.Context
import androidx.work.*
import java.util.concurrent.TimeUnit

class AndroidSyncScheduler(private val context: Context) : SyncScheduler {
    private val workManager get() = WorkManager.getInstance(context)
    private val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED).build()

    override fun scheduleOneTime(tag: String) {
        workManager.enqueue(
            OneTimeWorkRequestBuilder<SyncWorker>()
                .setConstraints(constraints).addTag(tag).build()
        )
    }

    override fun schedulePeriodic(tag: String, intervalMinutes: Long) {
        workManager.enqueueUniquePeriodicWork(tag, ExistingPeriodicWorkPolicy.KEEP,
            PeriodicWorkRequestBuilder<SyncWorker>(intervalMinutes, TimeUnit.MINUTES)
                .setConstraints(constraints).addTag(tag).build()
        )
    }

    override fun cancel(tag: String) { workManager.cancelAllWorkByTag(tag) }
}
```

---

## IosSyncScheduler (iosMain)

```kotlin
package {package_base}.core.sync

import platform.BackgroundTasks.*
import platform.Foundation.NSDate
import platform.Foundation.dateByAddingTimeInterval

class IosSyncScheduler : SyncScheduler {
    override fun scheduleOneTime(tag: String) {
        val request = BGProcessingTaskRequest(identifier = tag).apply {
            requiresNetworkConnectivity = true
        }
        BGTaskScheduler.sharedScheduler.submitTaskRequest(request, error = null)
    }

    override fun schedulePeriodic(tag: String, intervalMinutes: Long) {
        val request = BGAppRefreshTaskRequest(identifier = tag).apply {
            earliestBeginDate = NSDate().dateByAddingTimeInterval(intervalMinutes * 60.0)
        }
        BGTaskScheduler.sharedScheduler.submitTaskRequest(request, error = null)
    }

    override fun cancel(tag: String) {
        BGTaskScheduler.sharedScheduler.cancelTaskRequestWithIdentifier(tag)
    }
}
```

---

## SyncWorker (androidMain)

```kotlin
package {package_base}.core.sync

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import org.koin.core.component.KoinComponent
import org.koin.core.component.inject

class SyncWorker(
    context: Context, params: WorkerParameters,
) : CoroutineWorker(context, params), KoinComponent {
    private val syncRegistry: SyncRegistry by inject()

    override suspend fun doWork(): Result {
        val tag = tags.firstOrNull() ?: return Result.failure()
        return try {
            syncRegistry.get(tag)?.sync() ?: return Result.failure()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}
```

---

## Sync Use Case Pattern

Each feature provides a `Sync<Feature>UseCase` that coordinates the refresh:

```kotlin
package {package_base}.<feature>.domain.usecases

import {package_base}.<feature>.data.api.repository.<Feature>Repository
import {package_base}.core.sync.SyncUseCase
import {package_base}.core.sync.retryWithBackoff

class Sync<Feature>UseCase(
    private val repository: <Feature>Repository,
) : SyncUseCase {
    override suspend fun sync() {
        retryWithBackoff { repository.refresh<Feature>s().getOrThrow() }
    }
}
```

---

## Koin DI Modules

### Core syncModule (commonMain)

```kotlin
val syncModule = module { single { SyncRegistry() } }
```

### Platform syncModule — expect/actual

```kotlin
// commonMain
expect val platformSyncModule: Module

// androidMain
actual val platformSyncModule: Module = module {
    singleOf(::AndroidSyncScheduler) bind SyncScheduler::class
}

// iosMain
actual val platformSyncModule: Module = module {
    singleOf(::IosSyncScheduler) bind SyncScheduler::class
}
```

Add `syncModule` and `platformSyncModule` to AppModule `includes(...)`.

### Feature Sync Registration

```kotlin
val <feature>DomainModule = module {
    factoryOf(::Get<Feature>sUseCase)
    factoryOf(::Sync<Feature>UseCase)
    single { get<SyncRegistry>().register("<feature>_sync", get<Sync<Feature>UseCase>()) }
}
```

---

## build.gradle.kts Dependencies

```kotlin
// core/sync/build.gradle.kts
plugins { id("{package_base}.kmp.library") }

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.koin.core)
        }
        androidMain.dependencies {
            implementation(libs.work.runtime.ktx)
        }
    }
}
```

---

## iOS Setup (manual — output as next steps)

BGTaskScheduler requires registration at app launch in `AppDelegate.swift`:

```swift
BGTaskScheduler.shared.register(forTaskWithIdentifier: "<feature>_sync", using: nil) { task in
    self.handleSync(task: task as! BGProcessingTask)
}
```

`Info.plist` entry:

```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array><string><feature>_sync</string></array>
```

---

## Key Rules

- **Always use network constraints** — never schedule sync without `requiresNetworkConnectivity` / `NetworkType.CONNECTED`
- **Idempotent sync operations** — `refresh<Feature>s()` must be safe to call repeatedly without side effects
- **Handle partial failures** — commit successful items; don't roll back all on partial failure
- **Exponential backoff** — never use fixed-interval retries for network operations
- **One SyncWorker, many use cases** — route via tag + `SyncRegistry`, don't create one Worker per feature
- **iOS periodic refresh is approximate** — `earliestBeginDate` is a hint, not a guarantee
- **Test with `SyncScheduler` interface** — mock the scheduler in tests, never depend on platform APIs directly
