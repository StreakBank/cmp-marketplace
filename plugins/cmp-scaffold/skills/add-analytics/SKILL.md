---
name: add-analytics
description: Add cross-platform analytics and crash reporting with Firebase via a shared AnalyticsService interface, Koin DI, and TrackScreen composable
context: fork
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Analytics & Crash Reporting

Add cross-platform analytics and crash reporting to a CMP app using Firebase. See [analytics-patterns.md](../../references/analytics-patterns.md) for templates.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check Existing Setup

Glob for `**/AnalyticsService.kt` and grep for `FirebaseAnalytics` across the codebase. If analytics is already configured, report what exists and offer to add only the missing pieces (e.g., TrackScreen helper, crash reporting, FakeAnalyticsService).

### 3. Read Reference File

Read [analytics-patterns.md](../../references/analytics-patterns.md) for all templates — interface, platform implementations, DI module, TrackScreen helper, FakeAnalyticsService, and event naming conventions.

### 4. Add Version Catalog Entries

Add Firebase version catalog entries to `gradle/libs.versions.toml` if not already present. Add the `google-services` and `firebase-crashlytics` plugin entries.

### 5. Create AnalyticsService Interface

Create `core/analytics/src/commonMain/kotlin/{package_base_path}/core/analytics/AnalyticsService.kt` with the common interface: `logEvent`, `logScreenView`, `setUserProperty`, `logError`.

### 6. Create Platform Implementations

- **androidMain**: `AndroidAnalyticsService` using Firebase Analytics + Crashlytics SDKs
- **iosMain**: `IosAnalyticsService` using Firebase iOS SDK interop

### 7. Create TrackScreen Helper

Create `core/analytics/src/commonMain/kotlin/{package_base_path}/core/analytics/TrackScreen.kt` — a `@Composable` function using `LaunchedEffect` to log screen views. Uses `koinInject<AnalyticsService>()`.

### 8. Create AnalyticsEvents Constants

Create `core/analytics/src/commonMain/kotlin/{package_base_path}/core/analytics/AnalyticsEvents.kt` — an `object` with `const val` entries for each event. Scan existing features (glob for `*ViewModel.kt`) and generate initial event constants based on discovered feature names and actions.

### 9. Create FakeAnalyticsService

Create `core/analytics/src/commonMain/kotlin/{package_base_path}/core/analytics/FakeAnalyticsService.kt` with list-backed recording of events, screen views, errors, and user properties. Include `clear()` helper.

### 10. Create DI Module

Create expect/actual `analyticsPlatformModule` and common `analyticsModule`:
- **commonMain**: `expect val analyticsPlatformModule: Module` + `val analyticsModule`
- **androidMain**: `actual val` providing `FirebaseAnalytics.getInstance(get())`, `FirebaseCrashlytics.getInstance()`, and `AndroidAnalyticsService`
- **iosMain**: `actual val` providing `IosAnalyticsService`

Add `analyticsModule` to `AppModule.kt` `includes(...)`.

### 11. Register Module

Add `core:analytics` to `settings.gradle.kts` `include(...)`. Add `project(":core:analytics")` dependency to `composeApp/build.gradle.kts` and any feature modules that inject `AnalyticsService` directly.

### 12. Apply Gradle Plugins

Add `google-services` and `firebase-crashlytics` plugins to `composeApp/build.gradle.kts` plugins block. Verify `google-services.json` placement note for the user.

### 13. Wire TrackScreen into Existing Screens

Grep for `@Composable` screen functions in feature modules. Add `TrackScreen("<feature>_<screen>")` at the top of each screen composable found.

### 14. Wire Error Tracking into ViewModels

Grep for `_messages.trySend` or `onFailure` patterns in existing ViewModels. Where found, add `analytics.logError()` calls before the error message is sent — the raw throwable goes to `analytics.logError()` only, never into the `UiMessage` text itself (see [code-templates.md](../../references/code-templates.md#message-mapper-seam)). Add `AnalyticsService` as a constructor parameter to those ViewModels and update their Koin bindings.

### 15. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `AnalyticsService` interface in commonMain with all four methods
- [ ] `AndroidAnalyticsService` uses Firebase Analytics + Crashlytics (no direct `Log` calls)
- [ ] `IosAnalyticsService` uses Firebase iOS SDK interop
- [ ] `TrackScreen` composable uses `LaunchedEffect` with `screenName` key
- [ ] `FakeAnalyticsService` records all calls with list-backed fields
- [ ] `AnalyticsEvents` object uses `const val` with `snake_case` names
- [ ] Koin DI: expect/actual `analyticsPlatformModule` + common `analyticsModule`
- [ ] `analyticsModule` included in AppModule
- [ ] No PII in any event parameter values or constants
- [ ] All event name strings use constants, never inline literals

### 16. Report

Output a summary listing files created, files modified, and next steps:
- Place `google-services.json` in `composeApp/` (download from Firebase Console)
- Add `GoogleService-Info.plist` to the iOS Xcode project
- Call `FirebaseApp.configure()` in iOS `AppDelegate` before any analytics calls
- Add Firebase iOS SDK via CocoaPods (`pod 'FirebaseAnalytics'`, `pod 'FirebaseCrashlytics'`) or SPM
- Run `./gradlew assembleDebug` to verify Android builds
- `/cmp-scaffold:scaffold-tests` to generate tests verifying analytics calls via `FakeAnalyticsService`
- `/cmp-quality:review-changes` to validate all changes

## Troubleshooting

- `google-services.json` not found → download from Firebase Console → Project Settings → Android app → place in `composeApp/`
- iOS crash on launch → ensure `FirebaseApp.configure()` is called in `AppDelegate` before `ComposeUIViewController`
- Firebase Analytics events not appearing → events can take up to 24 hours in Firebase Console; use DebugView for real-time testing
- CocoaPods interop not resolving → run `pod install` in `iosApp/` and verify `cocoapods {}` block in `composeApp/build.gradle.kts`
- `FakeAnalyticsService` in production → verify Koin module order; `analyticsPlatformModule` should bind the real implementation, not the fake
