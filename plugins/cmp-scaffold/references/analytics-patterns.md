# Analytics & Crash Reporting Patterns

Cross-platform analytics and crash reporting for Compose Multiplatform via Firebase.

---

## Version Catalog Entries

```toml
[versions]
firebase-analytics-ktx = "22.4.0"
firebase-crashlytics-ktx = "19.4.1"
firebase-crashlytics-plugin = "3.0.3"
google-services = "4.4.2"

[libraries]
firebase-analytics-ktx = { group = "com.google.firebase", name = "firebase-analytics-ktx", version.ref = "firebase-analytics-ktx" }
firebase-crashlytics-ktx = { group = "com.google.firebase", name = "firebase-crashlytics-ktx", version.ref = "firebase-crashlytics-ktx" }

[plugins]
google-services = { id = "com.google.gms.google-services", version.ref = "google-services" }
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "firebase-crashlytics-plugin" }
```

> **Note:** Android uses Firebase Android SDK directly. iOS uses Firebase iOS SDK via CocoaPods or SPM — configured in the Xcode project, not in Gradle.

---

## AnalyticsService Interface (commonMain)

Place in `core/analytics/`:

```kotlin
package {package_base}.core.analytics

interface AnalyticsService {
    fun logEvent(name: String, params: Map<String, Any> = emptyMap())
    fun logScreenView(screenName: String)
    fun setUserProperty(key: String, value: String)
    fun logError(throwable: Throwable)
}
```

---

## Android Implementation (androidMain)

```kotlin
package {package_base}.core.analytics

import com.google.firebase.analytics.FirebaseAnalytics
import com.google.firebase.analytics.logEvent
import com.google.firebase.crashlytics.FirebaseCrashlytics

class AndroidAnalyticsService(
    private val firebaseAnalytics: FirebaseAnalytics,
    private val crashlytics: FirebaseCrashlytics,
) : AnalyticsService {

    override fun logEvent(name: String, params: Map<String, Any>) {
        firebaseAnalytics.logEvent(name) {
            params.forEach { (key, value) ->
                when (value) {
                    is String -> param(key, value)
                    is Long -> param(key, value)
                    is Double -> param(key, value)
                    else -> param(key, value.toString())
                }
            }
        }
    }

    override fun logScreenView(screenName: String) {
        firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
        }
    }

    override fun setUserProperty(key: String, value: String) =
        firebaseAnalytics.setUserProperty(key, value)

    override fun logError(throwable: Throwable) =
        crashlytics.recordException(throwable)
}
```

---

## iOS Implementation (iosMain)

Uses Firebase iOS SDK via CocoaPods interop. Adapt import paths if using SPM.

```kotlin
package {package_base}.core.analytics

import cocoapods.FirebaseAnalytics.FIRAnalytics
import cocoapods.FirebaseCrashlytics.FIRCrashlytics

class IosAnalyticsService : AnalyticsService {

    override fun logEvent(name: String, params: Map<String, Any>) =
        FIRAnalytics.logEventWithName(name, parameters = params as? Map<String, Any>)

    override fun logScreenView(screenName: String) =
        FIRAnalytics.logEventWithName("screen_view", parameters = mapOf("screen_name" to screenName))

    override fun setUserProperty(key: String, value: String) =
        FIRAnalytics.setUserPropertyString(value, forName = key)

    override fun logError(throwable: Throwable) {
        FIRCrashlytics.crashlytics().recordError(throwable as NSError)
    }
}
```

---

## TrackScreen Composable Helper

```kotlin
package {package_base}.core.analytics

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import org.koin.compose.koinInject

@Composable
fun TrackScreen(screenName: String) {
    val analytics = koinInject<AnalyticsService>()
    LaunchedEffect(screenName) {
        analytics.logScreenView(screenName)
    }
}
```

Usage: `TrackScreen("orders_list")` at the top of every screen composable.

---

## Error Tracking Integration

Wire analytics into the existing `errorEvents` SharedFlow pattern. Add `AnalyticsService` as a ViewModel constructor parameter:

```kotlin
fun refresh() {
    viewModelScope.launch {
        repository.refresh<Feature>s()
            .onFailure { error ->
                analytics.logError(error)
                _errorEvents.emit(error.message ?: "Failed to refresh")
            }
    }
}
```

---

## FakeAnalyticsService (Testing & Previews)

```kotlin
package {package_base}.core.analytics

class FakeAnalyticsService : AnalyticsService {
    val loggedEvents = mutableListOf<Pair<String, Map<String, Any>>>()
    val loggedScreenViews = mutableListOf<String>()
    val loggedErrors = mutableListOf<Throwable>()
    val userProperties = mutableMapOf<String, String>()

    override fun logEvent(name: String, params: Map<String, Any>) { loggedEvents.add(name to params) }
    override fun logScreenView(screenName: String) { loggedScreenViews.add(screenName) }
    override fun setUserProperty(key: String, value: String) { userProperties[key] = value }
    override fun logError(throwable: Throwable) { loggedErrors.add(throwable) }

    fun clear() { loggedEvents.clear(); loggedScreenViews.clear(); loggedErrors.clear(); userProperties.clear() }
}
```

---

## DI Module (Koin)

**expect (commonMain)** — `core/analytics/di/AnalyticsModule.kt`:

```kotlin
package {package_base}.core.analytics.di

import org.koin.core.module.Module
import org.koin.dsl.module

expect val analyticsPlatformModule: Module

val analyticsModule = module { includes(analyticsPlatformModule) }
```

**actual (androidMain):**

```kotlin
actual val analyticsPlatformModule: Module = module {
    single { FirebaseAnalytics.getInstance(get()) }
    single { FirebaseCrashlytics.getInstance() }
    single<AnalyticsService> { AndroidAnalyticsService(get(), get()) }
}
```

**actual (iosMain):**

```kotlin
actual val analyticsPlatformModule: Module = module {
    single<AnalyticsService> { IosAnalyticsService() }
}
```

Add `analyticsModule` to AppModule `includes(...)`.

---

## Event Naming Convention

Format: `{feature}_{action}_{target}` in `snake_case`.

| Event | Example |
|-------|---------|
| Tap action | `orders_tap_detail` |
| Add item | `cart_add_item` |
| Remove item | `favorites_remove_item` |
| Submit form | `checkout_submit_payment` |
| Search | `catalog_search_query` |
| Toggle | `settings_toggle_dark_mode` |

Use constants — never inline string literals:

```kotlin
package {package_base}.core.analytics

object AnalyticsEvents {
    const val ORDERS_TAP_DETAIL = "orders_tap_detail"
    const val CART_ADD_ITEM = "cart_add_item"
    // ... per-feature constants
}
```

---

## Key Rules

- **Never log PII** — no email, phone, name, address, or payment info in event params
- **Use constants** for all event names — define in `AnalyticsEvents` object, never inline strings
- **Always call `TrackScreen`** at the top of every screen composable
- **Log errors to Crashlytics** — wire `logError()` into every `onFailure` handler
- **Use `FakeAnalyticsService`** in tests and `@Preview` — never import Firebase in test sources
- **Keep params flat** — Firebase limits: 25 params per event, 40-char key, 100-char string value
- **snake_case only** — Firebase normalizes event names; mixed case causes duplicate events
- **iOS requires `FirebaseApp.configure()`** in `AppDelegate` before any analytics calls
