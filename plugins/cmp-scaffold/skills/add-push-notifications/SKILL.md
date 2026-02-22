---
name: add-push-notifications
description: Add cross-platform push notification support with FCM (Android) and APNs (iOS), including notification channels, permission handling, and deep link navigation
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Push Notifications

Add cross-platform push notification support using Firebase Cloud Messaging (Android) and APNs (iOS). See [push-notifications-patterns.md](../../references/push-notifications-patterns.md) for templates.

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check Existing Setup

Glob for `**/PushNotificationService.kt`, `**/FirebaseMessagingService*.kt`, and `**/notifications/**/*.kt`. If push notifications are already configured, report what exists and offer to add only missing pieces (e.g., deep link handling, permission request).

### 3. Read Reference Patterns

Read [push-notifications-patterns.md](../../references/push-notifications-patterns.md) for all templates: common interface, Android FCM service, iOS APNs bridge, deep link navigation, and DI module.

### 4. Create Common Interface

Create `core/notifications/src/commonMain/kotlin/{package_base_path}/core/notifications/`:
- `PushNotificationService.kt` — interface with `onTokenReceived(token)` and `onNotificationReceived(data)`
- `NotificationPayload.kt` — data class and `Map<String, String>.toNotificationPayload()` extension
- `DefaultPushNotificationService.kt` — default implementation that delegates token storage and payload parsing

### 5. Create Android FCM Implementation

In `composeApp/src/androidMain/kotlin/{package_base_path}/`:
- `AppFirebaseMessagingService.kt` — extends `FirebaseMessagingService`, injects `PushNotificationService` via Koin, builds and shows `NotificationCompat` notifications with deep link `PendingIntent`
- Update `ExampleApplication.kt` (or existing `Application` subclass) — add `createNotificationChannel()` in `onCreate()` using the channel ID `{resource_prefix}_general`
- `RequestNotificationPermission.kt` — composable that requests `POST_NOTIFICATIONS` on Android 13+

### 6. Update AndroidManifest.xml

Add to `composeApp/src/androidMain/AndroidManifest.xml`:
- `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />`
- `<service>` declaration for `AppFirebaseMessagingService` with `MESSAGING_EVENT` intent filter

### 7. Document iOS APNs Setup

Create `core/notifications/src/iosMain/kotlin/{package_base_path}/core/notifications/`:
- `IosPushBridge.kt` — top-level functions `onTokenReceived(token)` and `onNotificationTapped(route)` callable from Swift

Add a `// TODO:` block in the bridge file documenting the required Swift-side setup:
- Enable Push Notifications capability in Xcode
- Enable Remote notifications Background Mode
- Implement `AppDelegate` methods (`didRegisterForRemoteNotificationsWithDeviceToken`, `UNUserNotificationCenterDelegate`)
- Call Kotlin bridge functions from Swift delegates

### 8. Add Deep Link Navigation

Create `core/notifications/src/commonMain/kotlin/{package_base_path}/core/notifications/NotificationRouter.kt`:
- `NavController.handleNotificationRoute(route: String?)` extension
- Route map skeleton with `when` expression matching notification route prefixes to `@Serializable` Nav 2.x destinations
- Document how to add new route mappings as features are added

### 9. Wire DI Module

Create `core/notifications/src/commonMain/kotlin/{package_base_path}/core/notifications/di/NotificationsModule.kt`:
- `notificationsModule` providing `DefaultPushNotificationService` bound to `PushNotificationService`
- Add `notificationsModule` to `AppModule.kt` `includes(...)`

### 10. Add Dependencies

Add to `gradle/libs.versions.toml`:
- `firebase-messaging` version and library entry
- `google-services` plugin entry

Update `composeApp/build.gradle.kts`:
- Apply `google-services` plugin
- Add `firebase-messaging` to `androidMain.dependencies`

Update project-level `build.gradle.kts`:
- Add `google-services` plugin with `apply false`

Register `core:notifications` module in `settings.gradle.kts` and add `project(":core:notifications")` dependency to `composeApp/build.gradle.kts`.

### 11. Integrate Permission Request

Add `RequestNotificationPermission()` call in the root `App()` composable (or `MainActivity`) so the permission prompt appears on first launch (Android 13+ only).

### 12. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `PushNotificationService` interface exists in commonMain with `onTokenReceived` and `onNotificationReceived`
- [ ] `NotificationPayload` data class with `toNotificationPayload()` mapper
- [ ] Android: `AppFirebaseMessagingService` registered in `AndroidManifest.xml`
- [ ] Android: Notification channel created in `Application.onCreate()`
- [ ] Android: `POST_NOTIFICATIONS` permission declared in manifest
- [ ] Android: `RequestNotificationPermission` composable handles Android 13+ runtime permission
- [ ] Android: `google-services` plugin applied, `firebase-messaging` dependency added
- [ ] iOS: Bridge functions exist in iosMain for Swift interop
- [ ] Deep link: `handleNotificationRoute()` extension on `NavController`
- [ ] DI: `notificationsModule` registered in `AppModule.kt`
- [ ] Module: `core:notifications` registered in `settings.gradle.kts`
- [ ] No hardcoded strings in user-visible notification text templates
- [ ] No hardcoded version numbers — all via version catalog

### 13. Report

Output a summary listing files created, files modified, and next steps:
- Place `google-services.json` from Firebase Console in `composeApp/`
- Implement iOS `AppDelegate` in Swift using the documented pattern
- Enable Push Notifications and Remote notifications capabilities in Xcode
- Add route mappings to `NotificationRouter` as features are created
- Implement `TokenRepository` to persist and send device token to your backend
- `/cmp-scaffold:scaffold-tests` to generate test scaffolding for notification handling
- `/cmp-quality:review-changes` to validate the integration

## Troubleshooting

- `google-services.json` not found → build will fail; download from Firebase Console → Project Settings → Android app
- FCM `onMessageReceived` not called → ensure data payload is used (not notification payload) for foreground delivery
- Notifications not showing on Android 13+ → verify `POST_NOTIFICATIONS` permission is granted at runtime
- iOS token not received → verify Push Notifications capability is enabled in Xcode and provisioning profile includes push entitlement
- Deep link not navigating → verify route string matches the `when` mapping in `NotificationRouter` and NavController is accessible
