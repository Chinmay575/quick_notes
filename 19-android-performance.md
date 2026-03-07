# 19 — Android Performance

---

## Q1. What is ANR and how do you prevent it?

**Answer:**
**ANR (Application Not Responding)** occurs when the main thread is blocked for too long:
- **5 seconds** for input events (touch, key press)
- **10 seconds** for BroadcastReceiver
- **20 seconds** for Service (foreground)

**Prevention:**
```kotlin
// ❌ Network/DB on main thread
val users = api.fetchUsers() // blocks UI thread → ANR

// ✅ Use coroutines with appropriate dispatcher
viewModelScope.launch {
    val users = withContext(Dispatchers.IO) { api.fetchUsers() }
    _uiState.value = UiState.Success(users)
}

// ❌ Heavy computation on main thread
val sorted = hugeList.sortedBy { complexComparison(it) }

// ✅ Move to Default dispatcher
withContext(Dispatchers.Default) { hugeList.sortedBy { complexComparison(it) } }
```

**Detection:** Enable StrictMode during development:
```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(StrictMode.ThreadPolicy.Builder()
        .detectAll()
        .penaltyLog()
        .build())
}
```

---

## Q2. How do you detect and fix memory leaks?

**Answer:**

**Common leak sources:**
1. Static references to Activity/Context
2. Non-static inner classes holding Activity reference
3. Unregistered listeners/callbacks
4. Long-running tasks holding Activity reference
5. Singleton holding Activity context

```kotlin
// ❌ Leaks Activity
class Singleton {
    companion object {
        var context: Context? = null // holds Activity → leak!
    }
}

// ✅ Use Application context
class Singleton(private val context: Context) {
    companion object {
        fun init(context: Context) = Singleton(context.applicationContext)
    }
}

// ❌ Inner class leaks enclosing Activity
class MyActivity : Activity() {
    val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) { /* leaks MyActivity */ }
    }
}

// ✅ Use WeakReference or static class
class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
    private val activityRef = WeakReference(activity)
    override fun handleMessage(msg: Message) {
        activityRef.get()?.handleResult()
    }
}
```

**Tools:**
- **LeakCanary** — automatic leak detection in debug builds
- **Android Studio Profiler** → Memory tab → Heap dump
- **MAT (Memory Analyzer Tool)**

```groovy
// Add LeakCanary
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.14'
// That's it! It auto-detects leaks and shows notifications
```

---

## Q3. How do you optimize app startup time?

**Answer:**

```kotlin
// 1. Use App Startup library (lazy init)
class MyInitializer : Initializer<MySDK> {
    override fun create(context: Context): MySDK {
        return MySDK.init(context)
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// 2. Defer non-critical initialization
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // Critical only
        Hilt.init()

        // Defer non-critical
        Handler(Looper.getMainLooper()).post {
            Analytics.init() // runs after first frame
            CrashReporter.init()
        }
    }
}

// 3. Use Baseline Profiles (pre-compile hot paths)
// 4. Remove unused ContentProviders in manifest
// 5. Use SplashScreen API for proper splash
installSplashScreen()
```

**Measure:** `adb shell am start -W com.app/.MainActivity` → shows TotalTime.

---

## Q4. Explain ProGuard/R8.

**Answer:**

R8 (replacement for ProGuard) performs:
1. **Code shrinking** — removes unused classes, methods, fields
2. **Obfuscation** — renames to short names (a, b, c)
3. **Optimization** — inlines methods, removes dead branches
4. **Resource shrinking** — removes unused resources

```groovy
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

```
# proguard-rules.pro
# Keep data classes used with Gson/Moshi
-keep class com.app.data.model.** { *; }

# Keep Retrofit interfaces
-keep,allowobfuscation interface com.app.data.remote.** { *; }

# Keep Parcelable
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}
```

---

## Q5. How do you optimize RecyclerView performance?

**Answer:**

```kotlin
// 1. Use DiffUtil (never notifyDataSetChanged)
class MyDiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(old: Item, new: Item) = old.id == new.id
    override fun areContentsTheSame(old: Item, new: Item) = old == new
}

// 2. Fixed size
recyclerView.setHasFixedSize(true)

// 3. View pool for shared view types
recyclerView.recycledViewPool.setMaxRecycledViews(VIEW_TYPE, 20)

// 4. Prefetch
recyclerView.layoutManager = LinearLayoutManager(context).apply {
    initialPrefetchItemCount = 4
}

// 5. Avoid nested RecyclerViews — use ConcatAdapter
val concatAdapter = ConcatAdapter(headerAdapter, mainAdapter, footerAdapter)

// 6. Use ViewHolder pattern (automatic with RecyclerView.Adapter)
// 7. Avoid requestLayout() in onBindViewHolder
// 8. Use setItemViewCacheSize for off-screen caching
recyclerView.setItemViewCacheSize(20)
```

---

## Q6. What is Baseline Profiles?

**Answer:**
Baseline Profiles are a list of classes and methods that should be pre-compiled (AOT) during app installation.

```kotlin
// Generate with Macrobenchmark
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateProfile() {
        rule.collect("com.app") {
            pressHome()
            startActivityAndWait()
            // Navigate critical user journeys
            device.findObject(By.text("Login")).click()
            device.waitForIdle()
        }
    }
}
```

**Benefits:** 30-50% faster cold start, smoother initial scrolling, no jank on first launch.

---

## Q7. How do you reduce APK size?

**Answer:**

| Technique | Savings |
|-----------|---------|
| Enable R8/minification | 20-50% |
| `shrinkResources true` | 5-20% |
| Split APKs by ABI | 50%+ per APK |
| Use App Bundle (AAB) | Google Play optimizes |
| Use WebP instead of PNG | 25-50% per image |
| Remove unused libraries | Varies |
| Use vector drawables over bitmaps | 80%+ for icons |
| ProGuard/R8 obfuscation | 5-10% |
| `android.enableR8.fullMode=true` | Additional 5-10% |

```bash
# Analyze
./gradlew app:dependencies --configuration releaseRuntimeClasspath
# APK Analyzer in Android Studio: Build → Analyze APK
```
