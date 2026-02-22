# Deep Linking Patterns (Nav 2.x)

Deep link and App Link patterns for Compose Multiplatform Navigation 2.x with `@Serializable` type-safe routes.

---

## URI Scheme Convention

```
{app_scheme}://<feature>/{id}
```

- `{app_scheme}` — custom scheme derived from `rootProject.name` (e.g., `examplecmp`)
- Feature name = lowercase route segment (e.g., `orders`, `profile`)
- Path parameters match `@Serializable` route fields

Examples:
```
examplecmp://orders/abc-123
examplecmp://profile/user-42
https://example.com/orders/abc-123     ← App Link / Universal Link
```

---

## Deep Link on a Composable Route

Add `deepLinks` to the existing `composable<Route>` call inside the feature's `NavGraphBuilder.*Graph()`:

```kotlin
import androidx.navigation.navDeepLink
import androidx.navigation.compose.composable
import androidx.navigation.toRoute

@Serializable data class <Feature>DetailRoute(val id: String)

fun NavGraphBuilder.<feature>Graph(navController: NavController) {
    navigation<<Feature>GraphRoute>(startDestination = <Feature>Route::class) {
        composable<<Feature>Route> { <Feature>Screen() }
        composable<<Feature>DetailRoute>(
            deepLinks = listOf(
                navDeepLink<<Feature>DetailRoute>(
                    basePath = "{app_scheme}://<feature>"
                ),
                navDeepLink<<Feature>DetailRoute>(
                    basePath = "https://{app_domain}/<feature>"
                ),
            )
        ) { backStackEntry ->
            val route = backStackEntry.toRoute<<Feature>DetailRoute>()
            <Feature>DetailScreen(id = route.id)
        }
    }
}
```

> **Type-safe deep links:** `navDeepLink<Route>(basePath = ...)` auto-maps `@Serializable` fields to URI path/query segments. No manual `uriPattern` string needed.

---

## Deep Link on a Top-Level Route (No Path Params)

For routes without parameters, use a simple URI pattern:

```kotlin
@Serializable data object <Feature>Route

composable<<Feature>Route>(
    deepLinks = listOf(
        navDeepLink<<Feature>Route>(
            basePath = "{app_scheme}://<feature>"
        ),
    )
) { <Feature>Screen() }
```

---

## Android Configuration

### AndroidManifest.xml — Intent Filter

Add inside the `<activity>` tag that hosts the NavHost:

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="singleTask">

    <!-- Custom scheme deep link -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="{app_scheme}"
            android:host="<feature>" />
    </intent-filter>

    <!-- App Links (https) — requires assetlinks.json verification -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="{app_domain}"
            android:pathPrefix="/<feature>" />
    </intent-filter>
</activity>
```

### App Links Verification (assetlinks.json)

Host at `https://{app_domain}/.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "{package_base}.app",
    "sha256_cert_fingerprints": ["<YOUR_SHA256_FINGERPRINT>"]
  }
}]
```

---

## iOS Configuration

### URL Types (Info.plist) — Custom Scheme

Add to the iOS target's `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>{app_scheme}</string>
        </array>
        <key>CFBundleURLName</key>
        <string>{package_base}.app</string>
    </dict>
</array>
```

### Universal Links (apple-app-site-association)

Host at `https://{app_domain}/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "apps": [],
    "details": [{
      "appID": "<TEAM_ID>.{package_base}.app",
      "paths": ["/<feature>/*"]
    }]
  }
}
```

Enable **Associated Domains** capability in Xcode: `applinks:{app_domain}`.

---

## Handling Deep Links from Platform Entry Points

### Android — MainActivity

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val navController = rememberNavController()
            ExampleApp(navController = navController)
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // NavController.handleDeepLink() processes the intent automatically
        // when using singleTask launchMode — no manual handling required
    }
}
```

> **Key:** With `android:launchMode="singleTask"` and Nav 2.x `composable(deepLinks = ...)`, the framework handles intent routing automatically. No manual URI parsing needed.

### iOS — SceneDelegate / App

Deep link URLs arrive via the platform and are forwarded to Compose. Handle in Swift:

```swift
// In your SwiftUI App or SceneDelegate
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
    guard let url = URLContexts.first?.url else { return }
    // Forward to your Compose NavController via a shared callback or notification
}
```

> **Note:** iOS deep link forwarding to Compose NavController requires a bridge (e.g., shared `MutableStateFlow<String?>` or platform callback). This is a known CMP limitation — Nav 2.x `handleDeepLink()` is Android-only.

---

## Argument Validation

Always validate deep link arguments in the ViewModel — external URIs may contain invalid data:

```kotlin
class <Feature>DetailViewModel(
    savedStateHandle: SavedStateHandle,
    private val repository: <Feature>Repository,
) : ViewModel() {

    private val route = savedStateHandle.toRoute<<Feature>DetailRoute>()

    val uiState: StateFlow<<Feature>DetailUiState> = flow {
        if (route.id.isBlank()) {
            emit(<Feature>DetailUiState.Error(message = "Invalid ID"))
            return@flow
        }
        repository.get<Feature>(route.id)
            .fold(
                onSuccess = { item ->
                    if (item != null) emit(<Feature>DetailUiState.Success(item))
                    else emit(<Feature>DetailUiState.Error(message = "Not found"))
                },
                onFailure = { emit(<Feature>DetailUiState.Error(it.message ?: "Unknown error")) }
            )
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), <Feature>DetailUiState.Loading)
}
```

---

## Testing Deep Links

### Android (adb)

```bash
# Custom scheme
adb shell am start -a android.intent.action.VIEW \
    -d "{app_scheme}://<feature>/test-id-123" \
    {package_base}.app

# App Link (https)
adb shell am start -a android.intent.action.VIEW \
    -d "https://{app_domain}/<feature>/test-id-123" \
    {package_base}.app

# Verify App Links configuration
adb shell pm get-app-links {package_base}.app
```

### iOS (xcrun)

```bash
# Custom scheme (simulator)
xcrun simctl openurl booted "{app_scheme}://<feature>/test-id-123"

# Universal Link (simulator)
xcrun simctl openurl booted "https://{app_domain}/<feature>/test-id-123"
```

---

## Key Rules

1. **Type-safe deep links** — use `navDeepLink<Route>(basePath = ...)` with `@Serializable` routes; never construct `uriPattern` strings manually
2. **Validate arguments** — deep link IDs may be blank, malformed, or point to deleted resources; handle gracefully in ViewModel
3. **`singleTask` launch mode** — required on Android to prevent duplicate activity instances from deep links
4. **Custom scheme first** — start with `{app_scheme}://` for development; add `https://` App Links / Universal Links when domain verification is ready
5. **iOS bridge required** — Nav 2.x `handleDeepLink()` is Android-only; iOS needs a platform bridge to forward URLs to the NavController
6. **No hardcoded strings** — error messages in UI via `stringResource()`; deep link URIs use scheme/domain placeholders from build config or constants
7. **One intent-filter per host** — each feature gets its own `<intent-filter>` block in AndroidManifest; do not merge unrelated features into one filter
