# 30 — Android Extras: Permissions, Notifications, Gradle, Multi-Module, Coroutines Advanced

---

## Q1. Explain Android's runtime permission model.

**Answer:**

Android 6.0+ requires requesting dangerous permissions at runtime, not just in the manifest.

```kotlin
// Permission categories:
// Normal (auto-granted): INTERNET, VIBRATE, BLUETOOTH
// Dangerous (runtime): CAMERA, LOCATION, CONTACTS, STORAGE, MICROPHONE, PHONE

class CameraActivity : AppCompatActivity() {

    private val cameraPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) openCamera()
        else handleDenied()
    }

    private val multiplePermissionsLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val allGranted = permissions.values.all { it }
        if (allGranted) proceedWithFeature()
    }

    private fun checkAndRequestCamera() {
        when {
            // Already granted
            ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
                == PackageManager.PERMISSION_GRANTED -> openCamera()

            // Should show rationale (user denied before but didn't select "Don't ask again")
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                showRationaleDialog()
            }

            // First time or "Don't ask again" selected
            else -> cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private fun showRationaleDialog() {
        MaterialAlertDialogBuilder(this)
            .setTitle("Camera Permission Needed")
            .setMessage("We need camera access to take photos")
            .setPositiveButton("Grant") { _, _ ->
                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
            .setNegativeButton("Cancel", null)
            .show()
    }

    private fun handleDenied() {
        if (!shouldShowRequestPermissionRationale(Manifest.permission.CAMERA)) {
            // User selected "Don't ask again" — guide to settings
            showSettingsDialog()
        }
    }
}

// Android 13+ photo/video permissions:
// READ_MEDIA_IMAGES, READ_MEDIA_VIDEO, READ_MEDIA_AUDIO
// (replaces READ_EXTERNAL_STORAGE)

// Android 14+ partial photo access:
// READ_MEDIA_VISUAL_USER_SELECTED
```

---

## Q2. How do Notification Channels work in Android?

**Answer:**

```kotlin
// Required for Android 8.0+ (API 26+)
class NotificationHelper(private val context: Context) {

    companion object {
        const val CHANNEL_MESSAGES = "messages"
        const val CHANNEL_UPDATES = "updates"
        const val CHANNEL_PROMOTIONS = "promotions"
    }

    fun createChannels() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val messagesChannel = NotificationChannel(
                CHANNEL_MESSAGES, "Messages",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "Chat message notifications"
                enableLights(true)
                lightColor = Color.BLUE
                enableVibration(true)
                setShowBadge(true)
            }

            val updatesChannel = NotificationChannel(
                CHANNEL_UPDATES, "App Updates",
                NotificationManager.IMPORTANCE_DEFAULT
            ).apply { description = "App update notifications" }

            val promoChannel = NotificationChannel(
                CHANNEL_PROMOTIONS, "Promotions",
                NotificationManager.IMPORTANCE_LOW
            ).apply { description = "Promotional notifications" }

            val manager = context.getSystemService(NotificationManager::class.java)
            manager.createNotificationChannels(listOf(messagesChannel, updatesChannel, promoChannel))
        }
    }

    fun showNotification(title: String, body: String) {
        // Android 13+ requires POST_NOTIFICATIONS runtime permission
        val notification = NotificationCompat.Builder(context, CHANNEL_MESSAGES)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .setContentIntent(createPendingIntent())
            .addAction(R.drawable.ic_reply, "Reply", replyPendingIntent)
            .setStyle(NotificationCompat.BigTextStyle().bigText(body))
            .build()

        NotificationManagerCompat.from(context).notify(System.currentTimeMillis().toInt(), notification)
    }
}

// Importance levels:
// IMPORTANCE_HIGH   → heads-up notification + sound
// IMPORTANCE_DEFAULT → sound, no heads-up
// IMPORTANCE_LOW    → no sound
// IMPORTANCE_MIN    → no sound, no status bar
```

---

## Q3. How do you handle configuration changes and process death in Android?

**Answer:**

**Configuration changes** (screen rotation, locale, dark mode): Activity destroyed and recreated.

```kotlin
// Method 1: ViewModel (survives config changes, NOT process death)
class MyViewModel : ViewModel() {
    var counter = 0
    private val _data = MutableStateFlow<List<Item>>(emptyList())
    val data: StateFlow<List<Item>> = _data
}

// Method 2: SavedStateHandle (survives process death)
class MyViewModel(private val savedState: SavedStateHandle) : ViewModel() {
    var counter: Int
        get() = savedState.get<Int>("counter") ?: 0
        set(value) { savedState["counter"] = value }

    val searchQuery = savedState.getStateFlow("query", "")
}

// Method 3: onSaveInstanceState (for Activities/Fragments)
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putInt("scroll_position", recyclerView.computeVerticalScrollOffset())
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    savedInstanceState?.getInt("scroll_position")?.let { restoreScroll(it) }
}
```

**Survival summary:**

| Mechanism | Config Change | Process Death | App Killed |
|-----------|:---:|:---:|:---:|
| ViewModel | ✅ | ❌ | ❌ |
| SavedStateHandle | ✅ | ✅ | ❌ |
| onSaveInstanceState | ✅ | ✅ | ❌ |
| Room / DataStore | ✅ | ✅ | ✅ |

---

## Q4. Explain Gradle build system basics for Android.

**Answer:**

```groovy
// settings.gradle.kts (project-level)
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "MyApp"
include(":app", ":core", ":feature-auth")

// build.gradle.kts (app module)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")  // replaces kapt
}

android {
    namespace = "com.example.myapp"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
        debug {
            isDebuggable = true
            applicationIdSuffix = ".debug"
        }
    }

    productFlavors {
        flavorDimensions += "environment"
        create("dev") { applicationIdSuffix = ".dev"; buildConfigField("String", "BASE_URL", "\"https://dev-api.example.com\"") }
        create("prod") { buildConfigField("String", "BASE_URL", "\"https://api.example.com\"") }
    }

    buildFeatures { compose = true; buildConfig = true }
    composeOptions { kotlinCompilerExtensionVersion = "1.5.4" }
}

dependencies {
    implementation(libs.androidx.core.ktx)        // version catalog
    implementation(project(":core"))               // module dependency
    testImplementation(libs.junit)
    androidTestImplementation(libs.espresso.core)
    debugImplementation(libs.leak.canary)          // only in debug
    ksp(libs.hilt.compiler)                        // annotation processing
}
```

**Version Catalog** (`gradle/libs.versions.toml`):
```toml
[versions]
kotlin = "1.9.20"
compose-bom = "2024.02.00"

[libraries]
androidx-core-ktx = { module = "androidx.core:core-ktx", version = "1.12.0" }

[plugins]
android-application = { id = "com.android.application", version = "8.2.0" }
```

---

## Q5. Explain multi-module architecture in Android.

**Answer:**

```
project/
├── app/                    # Application module — wires everything together
├── core/
│   ├── common/             # Shared utilities, extensions
│   ├── data/               # Repositories implementations
│   ├── domain/             # Use cases, domain models, repository interfaces
│   ├── network/            # Retrofit, OkHttp, API services
│   ├── database/           # Room database, DAOs
│   ├── ui/                 # Design system, shared composables, theme
│   └── testing/            # Test utilities, fakes
├── feature/
│   ├── auth/               # Login, signup, forgot password
│   ├── home/               # Home screen
│   ├── profile/            # User profile
│   └── settings/           # App settings
└── build-logic/            # Convention plugins (shared build config)
```

**Benefits:**
- **Build speed**: Changed module = only that module recompiles
- **Encapsulation**: Features can't depend on each other directly
- **Team scalability**: Teams own specific modules
- **Reusability**: Core modules shared across features

```kotlin
// Dependency direction: app → feature → core (never reverse)
// feature/auth/build.gradle.kts
dependencies {
    implementation(project(":core:domain"))
    implementation(project(":core:ui"))
    implementation(project(":core:data"))
}
```

---

## Q6. Explain advanced Coroutines: structured concurrency, exception handling.

**Answer:**

```kotlin
// Structured concurrency: child coroutines are bound to parent scope
// If parent is cancelled, all children are cancelled

// SupervisorJob: failure of one child doesn't cancel siblings
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

// supervisorScope: failure of one child doesn't cancel others
suspend fun loadDashboard() = supervisorScope {
    val profile = async { fetchProfile() }     // if this fails...
    val feed = async { fetchFeed() }           // ...this still continues
    val notifications = async { fetchNotifications() }

    try {
        showDashboard(profile.await(), feed.await(), notifications.await())
    } catch (e: Exception) {
        // Handle individual failures
    }
}

// coroutineScope: failure of any child cancels ALL siblings
suspend fun transferMoney() = coroutineScope {
    val debit = async { debitAccount(from, amount) }
    val credit = async { creditAccount(to, amount) }
    // If either fails, both are cancelled (all-or-nothing)
    debit.await()
    credit.await()
}

// Exception handling
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("TAG", "Caught: ${exception.message}")
}
scope.launch(handler) { /* uncaught exceptions go to handler */ }

// launch vs async exception behavior:
// launch: propagates exception to parent (use CoroutineExceptionHandler)
// async:  stores exception, thrown on .await() (use try-catch)
```

---

## Q7. Explain Kotlin Flow operators.

**Answer:**

```kotlin
// Intermediate operators (cold, lazy)
flow.map { it.toUpperCase() }
flow.filter { it.length > 3 }
flow.take(5)
flow.distinctUntilChanged()
flow.debounce(300) // emit only after 300ms of no new values (search)
flow.flatMapLatest { searchApi(it) } // cancel previous, start new
flow.flatMapConcat { fetchDetails(it) } // sequential
flow.flatMapMerge { fetchItem(it) } // concurrent

// Terminal operators (trigger collection)
flow.collect { println(it) }
flow.first()
flow.toList()
flow.reduce { acc, value -> acc + value }
flow.fold(0) { acc, value -> acc + value }

// Combining flows
val combined = combine(flow1, flow2, flow3) { a, b, c -> Triple(a, b, c) }
val zipped = flow1.zip(flow2) { a, b -> Pair(a, b) } // pairs elements 1:1

// Search with debounce pattern
searchEditText.textChanges()
    .debounce(300)
    .distinctUntilChanged()
    .filter { it.length >= 2 }
    .flatMapLatest { query -> searchRepository(query) }
    .flowOn(Dispatchers.IO)
    .collect { results -> updateUI(results) }

// StateFlow vs SharedFlow
// StateFlow: always has value, replay=1, distinctUntilChanged
// SharedFlow: configurable replay, no initial value required
```

---

## Q8. What is the difference between APK and App Bundle (AAB)?

**Answer:**

| Aspect | APK | App Bundle (AAB) |
|--------|-----|-------------------|
| **Format** | Single installable file | Upload format for Play Store |
| **Size** | Contains ALL resources | Play Store generates optimized APKs |
| **Splitting** | No (or manual splits) | Automatic: language, density, ABI |
| **Average savings** | — | 15-20% smaller downloads |
| **Distribution** | Any (sideload, stores) | Google Play only (default) |
| **Signing** | You sign | Google Play App Signing |
| **Requirement** | — | Required for new Play Store apps since 2021 |

```bash
# Build APK (for testing/sideloading)
./gradlew assembleRelease

# Build AAB (for Play Store)
./gradlew bundleRelease

# Test AAB locally with bundletool
bundletool build-apks --bundle=app.aab --output=app.apks --ks=keystore.jks
bundletool install-apks --apks=app.apks
```
