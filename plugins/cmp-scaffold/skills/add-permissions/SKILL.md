---
name: add-permissions
description: Add cross-platform runtime permission handling (camera, location, microphone, photo-library, notifications) using expect/actual with native APIs
argument-hint: <permission-type>
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Add Runtime Permissions

Add cross-platform runtime permission handling to the project. See [permissions-patterns.md](../../references/permissions-patterns.md) for templates.

## Input

`$ARGUMENTS` — the permission type: `camera`, `location`, `microphone`, `photo-library`, or `notifications`

## Instructions

### 1. Detect Project Context

Read `settings.gradle.kts` → `rootProject.name` → lowercase = `{resource_prefix}`. Read `composeApp/build.gradle.kts` → `namespace` → `{package_base}` (strip `.app`). Derive `{package_base_path}` (dots → `/`).

### 2. Check for Existing Permissions Module

Glob for `**/core/permissions/**/*.kt`. If the `Permission` enum, `PermissionResult`, and `PermissionHandler` already exist, skip to Step 5.

### 3. Create Core Permissions Module

Create `core/permissions/` module with:
- `Permission` enum in commonMain
- `PermissionResult` sealed interface in commonMain
- `PermissionHandler` expect class in commonMain
- `PermissionRequester` expect class in commonMain
- Android actual implementations in androidMain (using `ActivityResultContracts.RequestPermission()`)
- iOS actual implementations in iosMain (using native frameworks: `AVFoundation`, `CoreLocation`, `Photos`, `UserNotifications`)
- `PermissionRequestEffect` composable in commonMain for rationale dialog flow
- Koin `permissionsModule` in commonMain

Use templates from [permissions-patterns.md](../../references/permissions-patterns.md).

### 4. Register Core Permissions Module

- Add `include(":core:permissions")` to `settings.gradle.kts`
- Create `core/permissions/build.gradle.kts` matching existing core module build patterns
- Add `permissionsModule` to AppModule `includes(...)`

### 5. Add Permission to Android Manifest

Read `composeApp/src/androidMain/AndroidManifest.xml`. Add the required `<uses-permission>` entry for the requested permission type. Add `<uses-feature>` with `android:required="false"` for hardware-dependent permissions (camera). Do not duplicate entries already present.

### 6. Document iOS Info.plist Changes

Output the `Info.plist` key and description string the user must add in Xcode. This cannot be automated from Kotlin — the user must add it manually to the iOS target.

### 7. Integrate into Feature Screen

If a feature screen is identifiable from context, add the `PermissionRequestEffect` composable with:
- Rationale title and message via `stringResource()`
- `onGranted` callback triggering the permission-gated action
- `onDenied` callback with appropriate fallback UX

### 8. Add String Resources

Add permission-related strings to the appropriate `strings.xml`:
- `permissions_<type>_rationale_title`
- `permissions_<type>_rationale_message`
- `permissions_action_grant`
- `permissions_action_deny`

### 9. Verify

Before reporting, confirm each item — fix any violations:

- [ ] `Permission` enum, `PermissionResult` sealed interface, `PermissionHandler` expect/actual all present
- [ ] Android actual uses `ActivityResultContracts.RequestPermission()` and `ContextCompat.checkSelfPermission()`
- [ ] iOS actual uses native frameworks (`AVCaptureDevice`, `CLLocationManager`, `PHPhotoLibrary`, or `UNUserNotificationCenter`)
- [ ] `<uses-permission>` entry added to `AndroidManifest.xml`
- [ ] iOS `Info.plist` key documented for user
- [ ] Koin `permissionsModule` registered in AppModule
- [ ] `core:permissions` module registered in `settings.gradle.kts`
- [ ] All user-visible strings via `stringResource()` — no hardcoded text
- [ ] Design tokens for any UI spacing — no raw dp

### 10. Report

Output a summary listing files created, files modified, and next steps:
- Add the documented `Info.plist` key in Xcode for iOS
- Handle `DeniedPermanently` by deep-linking to system Settings
- `/cmp-scaffold:scaffold-tests core` to generate tests for the permissions module
- `/cmp-scaffold:polish-ui <feature>` to enhance the permission-gated UI

## Troubleshooting

- `ActivityResultContracts` not resolved → ensure `composeApp` depends on `androidx.activity:activity-compose`
- iOS permission callback not firing → native callbacks run on main thread; dispatch to Compose coroutine context if updating state
- `POST_NOTIFICATIONS` permission crash on API < 33 → guard with `Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU`
- Permission already granted but still showing rationale → check `ContextCompat.checkSelfPermission()` before launching the request
- `core:permissions` module not found → verify `include(":core:permissions")` in `settings.gradle.kts` and sync Gradle
