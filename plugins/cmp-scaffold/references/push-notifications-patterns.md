# Push Notification Patterns (FCM + APNs)

Cross-platform push notification support using Firebase Cloud Messaging (Android) and APNs (iOS) with shared handling in commonMain.

---

## Version Catalog Entries

```toml
[versions]
firebase-messaging = "24.1.1"

[libraries]
firebase-messaging = { group = "com.google.firebase", name = "firebase-messaging-ktx", version.ref = "firebase-messaging" }

[plugins]
google-services = { id = "com.google.gms.google-services", version = "4.4.2" }
```

---

## Common Interface

Place in `core/notifications/` module:

```kotlin
package {package_base}.core.notifications

interface PushNotificationService {
    fun onTokenReceived(token: String)
    fun onNotificationReceived(data: Map<String, String>)
}

data class NotificationPayload(
    val route: String?,
    val title: String?,
    val body: String?,
)

fun Map<String, String>.toNotificationPayload() = NotificationPayload(
    route = get("route"), title = get("title"), body = get("body"),
)
```

---

## Android: Notification Channel

Create in `Application.onCreate()` — required for Android 8+:

```kotlin
private fun createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            "{resource_prefix}_general", "General",
            NotificationManager.IMPORTANCE_DEFAULT,
        )
        getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
    }
}
```

---

## Android: FirebaseMessagingService

```kotlin
package {package_base}

class AppFirebaseMessagingService : FirebaseMessagingService() {
    private val pushService: PushNotificationService by inject()

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        pushService.onTokenReceived(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)
        pushService.onNotificationReceived(message.data)
        val title = message.data["title"] ?: message.notification?.title ?: return
        val body = message.data["body"] ?: message.notification?.body ?: ""
        showNotification(title, body, message.data["route"])
    }

    private fun showNotification(title: String, body: String, route: String?) {
        val intent = packageManager.getLaunchIntentForPackage(packageName)?.apply {
            flags = Intent.FLAG_ACTIVITY_SINGLE_TOP
            route?.let { putExtra("route", it) }
        }
        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )
        val notification = NotificationCompat.Builder(this, "{resource_prefix}_general")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentTitle(title).setContentText(body)
            .setAutoCancel(true).setContentIntent(pendingIntent).build()
        getSystemService(NotificationManager::class.java)
            .notify(System.currentTimeMillis().toInt(), notification)
    }
}
```

---

## Android: Manifest Entries

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service android:name=".AppFirebaseMessagingService" android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

---

## Android: POST_NOTIFICATIONS Permission (Android 13+)

```kotlin
@Composable
fun RequestNotificationPermission() {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) return
    val context = LocalContext.current
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { /* granted: Boolean */ }
    LaunchedEffect(Unit) {
        if (ContextCompat.checkSelfPermission(
                context, Manifest.permission.POST_NOTIFICATIONS
            ) != PackageManager.PERMISSION_GRANTED
        ) { launcher.launch(Manifest.permission.POST_NOTIFICATIONS) }
    }
}
```

---

## iOS: APNs AppDelegate (Swift)

```swift
class AppDelegate: NSObject, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { _, _ in }
        application.registerForRemoteNotifications()
        return true
    }

    func application(_ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        MainViewControllerKt.onTokenReceived(token: token)
    }

    func userNotificationCenter(_ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler handler: @escaping (UNNotificationPresentationOptions) -> Void) {
        handler([.banner, .badge, .sound])
    }

    func userNotificationCenter(_ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler handler: @escaping () -> Void) {
        if let route = response.notification.request.content.userInfo["route"] as? String {
            MainViewControllerKt.onNotificationTapped(route: route)
        }
        handler()
    }
}
```

---

## iOS: Kotlin Bridge (iosMain)

```kotlin
package {package_base}.core.notifications

fun onTokenReceived(token: String) {
    org.koin.core.context.GlobalContext.get().get<PushNotificationService>()
        .onTokenReceived(token)
}

fun onNotificationTapped(route: String) {
    org.koin.core.context.GlobalContext.get().get<PushNotificationService>()
        .onNotificationReceived(mapOf("route" to route))
}
```

---

## Deep Link Navigation

```kotlin
package {package_base}.core.notifications

import androidx.navigation.NavController

fun NavController.handleNotificationRoute(route: String?) {
    if (route.isNullOrBlank()) return
    val segments = route.split("/")
    try {
        when (segments.firstOrNull()) {
            // Map routes → Nav 2.x @Serializable destinations
            // e.g. "orders" → navigate(OrderDetailRoute(id = segments[1]))
            else -> { /* unknown route */ }
        }
    } catch (_: Exception) { /* malformed route */ }
}
```

Android: extract route from intent in `MainActivity`: `intent?.getStringExtra("route")`.

---

## Koin DI Module

```kotlin
package {package_base}.core.notifications.di

import {package_base}.core.notifications.*
import org.koin.dsl.module

val notificationsModule = module {
    single<PushNotificationService> { DefaultPushNotificationService(tokenRepository = get()) }
}
```

```kotlin
class DefaultPushNotificationService(
    private val tokenRepository: TokenRepository,
) : PushNotificationService {
    override fun onTokenReceived(token: String) { tokenRepository.saveToken(token) }
    override fun onNotificationReceived(data: Map<String, String>) {
        val payload = data.toNotificationPayload()
        // Handle in-app display or deep link routing
    }
}
```

Add `notificationsModule` to AppModule `includes(...)`.

---

## build.gradle.kts Dependencies

```kotlin
// composeApp/build.gradle.kts
plugins { alias(libs.plugins.google.services) }
kotlin {
    sourceSets {
        androidMain.dependencies { implementation(libs.firebase.messaging) }
    }
}

// Project-level build.gradle.kts
plugins { alias(libs.plugins.google.services) apply false }
```

> **Important:** Place `google-services.json` in `composeApp/` (from Firebase Console).

---

## Key Rules

- **Always create notification channels** on Android 8+ — notifications silently dropped without them
- **Handle both foreground and background** — use data payloads (`"data": {...}`), not notification payloads, for consistent cross-platform delivery
- **Send token to backend on every refresh** — tokens rotate; `onNewToken` is the canonical callback
- **Request POST_NOTIFICATIONS** on Android 13+ before showing notifications
- **iOS requires capabilities** — enable "Push Notifications" and "Remote notifications" Background Mode in Xcode
- **Deep link routes must match** `@Serializable` Nav 2.x destinations — keep route map in sync
