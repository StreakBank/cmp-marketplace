# Permissions Patterns (expect/actual)

Cross-platform runtime permission handling for Compose Multiplatform using expect/actual with native APIs.

---

## Permission Types

```kotlin
package {package_base}.core.permissions

enum class Permission {
    Camera,
    Location,
    Microphone,
    PhotoLibrary,
    Notifications,
}
```

---

## Permission Result

```kotlin
package {package_base}.core.permissions

sealed interface PermissionResult {
    data object Granted : PermissionResult
    data object Denied : PermissionResult
    data object DeniedPermanently : PermissionResult
}
```

---

## PermissionHandler (expect/actual)

### expect (commonMain)

```kotlin
package {package_base}.core.permissions

import androidx.compose.runtime.Composable

expect class PermissionHandler {
    @Composable
    fun rememberPermissionState(
        permission: Permission,
        onResult: (PermissionResult) -> Unit,
    ): PermissionRequester
}

expect class PermissionRequester {
    fun launch()
}
```

### actual (androidMain)

```kotlin
package {package_base}.core.permissions

import android.content.pm.PackageManager
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
import androidx.core.content.ContextCompat

actual class PermissionHandler {
    @Composable
    actual fun rememberPermissionState(
        permission: Permission,
        onResult: (PermissionResult) -> Unit,
    ): PermissionRequester {
        val context = LocalContext.current
        val manifestPermission = permission.toAndroidPermission()
        val launcher = rememberLauncherForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { granted ->
            if (granted) {
                onResult(PermissionResult.Granted)
            } else {
                onResult(PermissionResult.Denied)
            }
        }
        return remember(permission) {
            PermissionRequester {
                val already = ContextCompat.checkSelfPermission(
                    context, manifestPermission
                ) == PackageManager.PERMISSION_GRANTED
                if (already) {
                    onResult(PermissionResult.Granted)
                } else {
                    launcher.launch(manifestPermission)
                }
            }
        }
    }
}

actual class PermissionRequester(private val action: () -> Unit) {
    actual fun launch() = action()
}

private fun Permission.toAndroidPermission(): String = when (this) {
    Permission.Camera -> android.Manifest.permission.CAMERA
    Permission.Location -> android.Manifest.permission.ACCESS_FINE_LOCATION
    Permission.Microphone -> android.Manifest.permission.RECORD_AUDIO
    Permission.PhotoLibrary -> android.Manifest.permission.READ_MEDIA_IMAGES
    Permission.Notifications -> android.Manifest.permission.POST_NOTIFICATIONS
}
```

### actual (iosMain)

```kotlin
package {package_base}.core.permissions

import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import platform.AVFoundation.AVAuthorizationStatusAuthorized
import platform.AVFoundation.AVAuthorizationStatusDenied
import platform.AVFoundation.AVCaptureDevice
import platform.AVFoundation.AVMediaTypeAudio
import platform.AVFoundation.AVMediaTypeVideo
import platform.AVFoundation.authorizationStatusForMediaType
import platform.AVFoundation.requestAccessForMediaType
import platform.CoreLocation.CLAuthorizationStatus
import platform.CoreLocation.CLLocationManager
import platform.CoreLocation.kCLAuthorizationStatusAuthorizedWhenInUse
import platform.CoreLocation.kCLAuthorizationStatusDenied
import platform.Photos.PHAuthorizationStatusAuthorized
import platform.Photos.PHAuthorizationStatusDenied
import platform.Photos.PHPhotoLibrary
import platform.UserNotifications.UNAuthorizationOptionAlert
import platform.UserNotifications.UNAuthorizationOptionBadge
import platform.UserNotifications.UNAuthorizationOptionSound
import platform.UserNotifications.UNUserNotificationCenter

actual class PermissionHandler {
    @Composable
    actual fun rememberPermissionState(
        permission: Permission,
        onResult: (PermissionResult) -> Unit,
    ): PermissionRequester = remember(permission) {
        PermissionRequester { requestIosPermission(permission, onResult) }
    }
}

actual class PermissionRequester(private val action: () -> Unit) {
    actual fun launch() = action()
}

private fun requestIosPermission(
    permission: Permission,
    onResult: (PermissionResult) -> Unit,
) {
    when (permission) {
        Permission.Camera -> requestAvMediaPermission(AVMediaTypeVideo, onResult)
        Permission.Microphone -> requestAvMediaPermission(AVMediaTypeAudio, onResult)
        Permission.Location -> requestLocationPermission(onResult)
        Permission.PhotoLibrary -> requestPhotoLibraryPermission(onResult)
        Permission.Notifications -> requestNotificationPermission(onResult)
    }
}

private fun requestAvMediaPermission(
    mediaType: String,
    onResult: (PermissionResult) -> Unit,
) {
    val status = AVCaptureDevice.authorizationStatusForMediaType(mediaType)
    when (status) {
        AVAuthorizationStatusAuthorized -> onResult(PermissionResult.Granted)
        AVAuthorizationStatusDenied -> onResult(PermissionResult.DeniedPermanently)
        else -> AVCaptureDevice.requestAccessForMediaType(mediaType) { granted ->
            onResult(if (granted) PermissionResult.Granted else PermissionResult.Denied)
        }
    }
}

private fun requestLocationPermission(onResult: (PermissionResult) -> Unit) {
    val manager = CLLocationManager()
    when (manager.authorizationStatus) {
        kCLAuthorizationStatusAuthorizedWhenInUse -> onResult(PermissionResult.Granted)
        kCLAuthorizationStatusDenied -> onResult(PermissionResult.DeniedPermanently)
        else -> {
            manager.requestWhenInUseAuthorization()
            onResult(PermissionResult.Denied) // observe delegate for real status
        }
    }
}

private fun requestPhotoLibraryPermission(onResult: (PermissionResult) -> Unit) {
    val status = PHPhotoLibrary.authorizationStatus()
    when (status) {
        PHAuthorizationStatusAuthorized -> onResult(PermissionResult.Granted)
        PHAuthorizationStatusDenied -> onResult(PermissionResult.DeniedPermanently)
        else -> PHPhotoLibrary.requestAuthorization { newStatus ->
            onResult(
                if (newStatus == PHAuthorizationStatusAuthorized)
                    PermissionResult.Granted else PermissionResult.Denied
            )
        }
    }
}

private fun requestNotificationPermission(onResult: (PermissionResult) -> Unit) {
    val center = UNUserNotificationCenter.currentNotificationCenter()
    center.requestAuthorizationWithOptions(
        UNAuthorizationOptionAlert or UNAuthorizationOptionBadge or UNAuthorizationOptionSound
    ) { granted, _ ->
        onResult(if (granted) PermissionResult.Granted else PermissionResult.Denied)
    }
}
```

---

## Permission Request Composable

Reusable composable that handles the request-rationale-settings flow:

```kotlin
package {package_base}.core.permissions.ui

import androidx.compose.material3.AlertDialog
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import {package_base}.core.permissions.Permission
import {package_base}.core.permissions.PermissionHandler
import {package_base}.core.permissions.PermissionResult

@Composable
fun PermissionRequestEffect(
    permissionHandler: PermissionHandler,
    permission: Permission,
    rationaleTitle: String,
    rationaleMessage: String,
    onGranted: () -> Unit,
    onDenied: () -> Unit = {},
) {
    var showRationale by remember { mutableStateOf(false) }
    val requester = permissionHandler.rememberPermissionState(permission) { result ->
        when (result) {
            PermissionResult.Granted -> onGranted()
            PermissionResult.Denied -> showRationale = true
            PermissionResult.DeniedPermanently -> onDenied()
        }
    }

    if (showRationale) {
        AlertDialog(
            onDismissRequest = { showRationale = false },
            title = { Text(rationaleTitle) },
            text = { Text(rationaleMessage) },
            confirmButton = {
                TextButton(onClick = {
                    showRationale = false
                    requester.launch()
                }) {
                    Text(stringResource(Res.string.permissions_action_grant))
                }
            },
            dismissButton = {
                TextButton(onClick = {
                    showRationale = false
                    onDenied()
                }) {
                    Text(stringResource(Res.string.permissions_action_deny))
                }
            },
        )
    }

    requester.launch()
}
```

---

## Android Manifest Permissions

Add to `composeApp/src/androidMain/AndroidManifest.xml`:

```xml
<!-- Camera -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="false" />

<!-- Location -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Microphone -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />

<!-- Photo Library (API 33+) -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

<!-- Notifications (API 33+) -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

---

## iOS Info.plist Keys

Add to the Xcode project's `Info.plist`:

```xml
<!-- Camera -->
<key>NSCameraUsageDescription</key>
<string>This app needs camera access to take photos.</string>

<!-- Location -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>This app needs your location to show nearby results.</string>

<!-- Microphone -->
<key>NSMicrophoneUsageDescription</key>
<string>This app needs microphone access to record audio.</string>

<!-- Photo Library -->
<key>NSPhotoLibraryUsageDescription</key>
<string>This app needs photo library access to select images.</string>
```

> **Note:** Notifications do not require an Info.plist key. Replace the placeholder strings with descriptions specific to the app's use case.

---

## Koin DI Module

```kotlin
package {package_base}.core.permissions.di

import {package_base}.core.permissions.PermissionHandler
import org.koin.dsl.module

val permissionsModule = module {
    factory { PermissionHandler() }
}
```

Add `permissionsModule` to AppModule `includes(...)`.

---

## Key Rules

- Always provide a rationale dialog before re-requesting a denied permission
- Use `stringResource()` for all user-visible text in rationale dialogs — never hardcoded strings
- Use design tokens (`Spacing.*`) for dialog padding — no raw dp
- Android: declare permissions in `AndroidManifest.xml` before requesting at runtime
- iOS: add all required `NS*UsageDescription` keys to `Info.plist` — missing keys cause a crash
- Handle `DeniedPermanently` by guiding the user to system Settings
- Request permissions lazily (at point of use), not on app launch
- `POST_NOTIFICATIONS` and `READ_MEDIA_IMAGES` require API 33+ — guard with SDK version checks on older targets
