# 13 — Android Core Components

---

## Q1. What are the four main Android components?

**Answer:**

| Component | Purpose | Lifecycle |
|-----------|---------|-----------|
| **Activity** | Single screen with UI | Created → Started → Resumed → Paused → Stopped → Destroyed |
| **Service** | Background processing | Started service or Bound service |
| **Broadcast Receiver** | Responds to system-wide events | onReceive() |
| **Content Provider** | Shares data between apps | CRUD operations via URI |

All must be declared in `AndroidManifest.xml`.

---

## Q2. Explain the Activity lifecycle in detail.

**Answer:**

```
onCreate()  → first time, set up UI (setContentView)
  ↓
onStart()   → visible but not interactive
  ↓
onResume()  → foreground, interactive
  ↓
[User navigates away]
  ↓
onPause()   → partially visible (dialog, multi-window)
  ↓
onStop()    → not visible
  ↓
onDestroy() → being destroyed (finish() or config change)

onRestart() → called between onStop() and onStart() when returning
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // Initialize: ViewBinding, ViewModel, adapters
        // Restore state from savedInstanceState
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putString("key", value) // save transient state
    }

    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        value = savedInstanceState.getString("key")
    }
}
```

---

## Q3. Explain the Fragment lifecycle.

**Answer:**

```
onAttach()          → fragment attached to activity
onCreate()          → fragment created (no UI yet)
onCreateView()      → inflate layout
onViewCreated()     → view is ready, set up UI
onStart()           → visible
onResume()          → interactive
onPause()           → partially hidden
onStop()            → hidden
onDestroyView()     → view destroyed (fragment may still exist)
onDestroy()         → fragment destroyed
onDetach()          → detached from activity
```

```kotlin
class UserFragment : Fragment(R.layout.fragment_user) {
    private var _binding: FragmentUserBinding? = null
    private val binding get() = _binding!!
    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentUserBinding.bind(view)

        viewModel.users.observe(viewLifecycleOwner) { users ->
            adapter.submitList(users)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // avoid memory leaks
    }
}
```

---

## Q4. What is the difference between Activity and Fragment?

**Answer:**

| Feature | Activity | Fragment |
|---------|----------|----------|
| Lifecycle | Independent | Depends on host Activity |
| Back stack | System-managed | FragmentManager-managed |
| UI | Full screen | Part of screen (reusable) |
| Communication | Intent | ViewModel, interfaces, FragmentResult |
| Navigation | startActivity() | FragmentTransaction / Navigation Component |

**Modern approach:** Single Activity + multiple Fragments using Navigation Component.

---

## Q5. Explain Services — Started vs Bound vs Foreground.

**Answer:**

```kotlin
// Started Service — runs independently until stopped
class DownloadService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Do work on background thread
        return START_STICKY // restart if killed
    }
    override fun onBind(intent: Intent?): IBinder? = null
}

// Bound Service — client-server interface
class MusicService : Service() {
    private val binder = MusicBinder()
    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }
    override fun onBind(intent: Intent?): IBinder = binder
}

// Foreground Service — must show notification (Android 8+)
class LocationService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        startForeground(NOTIFICATION_ID, notification)
        // Track location
        return START_STICKY
    }
}
```

| Type | Notification | Lifecycle | Use Case |
|------|-------------|-----------|----------|
| Started | No | Runs until stopSelf() | One-time operations |
| Bound | No | Lives with bound clients | IPC, music player controls |
| Foreground | Required | Long-running | Location tracking, music playback |

---

## Q6. What is an Intent? Explain implicit vs explicit intents.

**Answer:**

```kotlin
// Explicit Intent — target component is specified
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("USER_ID", 42)
startActivity(intent)

// Implicit Intent — system finds matching component
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://google.com"))
startActivity(intent)

val intent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Share this!")
}
startActivity(Intent.createChooser(intent, "Share via"))

// Intent filters (AndroidManifest.xml)
<activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

---

## Q7. Explain Broadcast Receivers.

**Answer:**

```kotlin
// Dynamic registration
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_BATTERY_LOW -> showLowBatteryWarning()
            ConnectivityManager.CONNECTIVITY_ACTION -> checkNetwork()
        }
    }
}

// Register dynamically
val receiver = MyReceiver()
val filter = IntentFilter().apply {
    addAction(Intent.ACTION_BATTERY_LOW)
    addAction(ConnectivityManager.CONNECTIVITY_ACTION)
}
registerReceiver(receiver, filter)

// Unregister (avoid leaks!)
unregisterReceiver(receiver)

// Local broadcasts (within app only)
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)
```

**Note:** Many implicit broadcasts are restricted since Android 8.0. Use WorkManager for background work.

---

## Q8. What is a Content Provider?

**Answer:**

```kotlin
// Querying contacts (using Content Provider)
val cursor = contentResolver.query(
    ContactsContract.Contacts.CONTENT_URI,
    arrayOf(ContactsContract.Contacts.DISPLAY_NAME),
    null, null,
    ContactsContract.Contacts.DISPLAY_NAME + " ASC"
)

cursor?.use {
    while (it.moveToNext()) {
        val name = it.getString(it.getColumnIndexOrThrow(ContactsContract.Contacts.DISPLAY_NAME))
        println(name)
    }
}

// Custom Content Provider
class MyProvider : ContentProvider() {
    override fun query(uri: Uri, projection: Array<String>?, selection: String?,
                      selectionArgs: Array<String>?, sortOrder: String?): Cursor? { /* ... */ }
    override fun insert(uri: Uri, values: ContentValues?): Uri? { /* ... */ }
    override fun update(uri: Uri, values: ContentValues?, selection: String?,
                       selectionArgs: Array<String>?): Int { /* ... */ }
    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int { /* ... */ }
    override fun getType(uri: Uri): String? { /* ... */ }
    override fun onCreate(): Boolean { /* ... */ }
}
```

---

## Q9. Explain Android app launch process.

**Answer:**

1. **User taps app icon** → Launcher sends Intent
2. **Zygote process** forks a new process for the app
3. **Application class** is created (custom `Application` subclass)
4. **ActivityThread** creates the main (UI) thread and Looper
5. **Activity** specified in the launcher intent-filter is created
6. **onCreate()** → `setContentView()` inflates the layout
7. **Window** is created, view hierarchy is measured/laid out/drawn
8. **First frame** is rendered

**Cold start vs Warm start vs Hot start:**
- **Cold:** App not in memory → full launch (slowest)
- **Warm:** App process exists but Activity destroyed → partial launch
- **Hot:** App in background → just onResume() (fastest)

---

## Q10. What is the Android Application class?

**Answer:**

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // Initialize once for entire app lifecycle:
        // - Dependency injection (Hilt, Koin)
        // - Firebase, analytics
        // - Crash reporting (Crashlytics)
        // - Timber for logging
        // - App-wide singletons
    }

    override fun onLowMemory() {
        super.onLowMemory()
        // Clear caches
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        // Release resources based on level
    }
}
```

Must be declared in manifest: `<application android:name=".MyApp">`
