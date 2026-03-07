# 14 — Android Jetpack Components

---

## Q1. What is Android Jetpack?

**Answer:**
Jetpack is a suite of libraries and tools that help build robust, maintainable, testable Android apps.

**Key categories:**
- **Architecture:** ViewModel, LiveData, Room, DataStore, Navigation, WorkManager
- **UI:** Compose, RecyclerView, Fragment, ViewPager2, ConstraintLayout
- **Behavior:** Notifications, Permissions, Camera, Media
- **Foundation:** AppCompat, Core KTX, Multidex

---

## Q2. Explain ViewModel.

**Answer:**

```kotlin
class UserViewModel(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // StateFlow (preferred over LiveData with Compose)
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    // SavedStateHandle survives process death
    val searchQuery = savedStateHandle.getStateFlow("query", "")

    init { loadUsers() }

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val users = repository.getUsers()
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown")
            }
        }
    }
}

// ViewModel survives configuration changes (rotation) but NOT process death
// SavedStateHandle survives process death

// In Activity/Fragment
val viewModel: UserViewModel by viewModels()

// With Hilt
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel()
```

---

## Q3. Explain LiveData vs StateFlow vs SharedFlow.

**Answer:**

| Feature | LiveData | StateFlow | SharedFlow |
|---------|----------|-----------|------------|
| Lifecycle-aware | Yes | No (need `repeatOnLifecycle`) | No |
| Initial value | No | Required | No |
| Replays | Latest value | Latest value | Configurable |
| Null values | Allowed | Allowed | Allowed |
| Platform | Android only | Kotlin (multiplatform) | Kotlin |
| Backpressure | Drops | Conflated (latest only) | Configurable |

```kotlin
// LiveData
val users: LiveData<List<User>> = repository.getUsers().asLiveData()

// Observe
viewModel.users.observe(viewLifecycleOwner) { users -> adapter.submitList(users) }

// StateFlow (preferred)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// Collect safely
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> updateUI(state) }
    }
}

// SharedFlow (events — one-time, NOT replayed)
private val _events = MutableSharedFlow<UiEvent>()
val events: SharedFlow<UiEvent> = _events.asSharedFlow()
```

---

## Q4. Explain Room database.

**Answer:**

```kotlin
// Entity
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "full_name") val name: String,
    val email: String,
    @ColumnInfo(defaultValue = "CURRENT_TIMESTAMP") val createdAt: String? = null
)

// DAO
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY full_name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>  // reactive

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: Int): UserEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: UserEntity)

    @Update
    suspend fun update(user: UserEntity)

    @Delete
    suspend fun delete(user: UserEntity)

    @Query("DELETE FROM users")
    suspend fun deleteAll()
}

// Database
@Database(entities = [UserEntity::class], version = 2, exportSchema = true)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// Build
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()

// Migration
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN avatar TEXT")
    }
}
```

---

## Q5. Explain Navigation Component.

**Answer:**

```xml
<!-- nav_graph.xml -->
<navigation xmlns:android="..."
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.app.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.app.DetailFragment"
        android:label="Detail">
        <argument
            android:name="userId"
            app:argType="integer" />
    </fragment>
</navigation>
```

```kotlin
// Navigate
findNavController().navigate(R.id.action_home_to_detail, bundleOf("userId" to 42))

// Safe Args (type-safe)
val action = HomeFragmentDirections.actionHomeToDetail(userId = 42)
findNavController().navigate(action)

// Receive args
val args: DetailFragmentArgs by navArgs()
val userId = args.userId
```

---

## Q6. Explain WorkManager.

**Answer:**

```kotlin
// Define work
class SyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            val data = inputData.getString("key")
            repository.sync(data)
            Result.success(workDataOf("result" to "done"))
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}

// Enqueue
val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build())
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .setInputData(workDataOf("key" to "value"))
    .build()

WorkManager.getInstance(context).enqueue(request)

// Periodic work
val periodicWork = PeriodicWorkRequestBuilder<SyncWorker>(15, TimeUnit.MINUTES)
    .build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync", ExistingPeriodicWorkPolicy.KEEP, periodicWork
)

// Chain work
WorkManager.getInstance(context)
    .beginWith(downloadWork)
    .then(processWork)
    .then(uploadWork)
    .enqueue()

// Observe
WorkManager.getInstance(context).getWorkInfoByIdLiveData(request.id)
    .observe(this) { info ->
        if (info.state == WorkInfo.State.SUCCEEDED) {
            val result = info.outputData.getString("result")
        }
    }
```

---

## Q7. What is DataStore (replacement for SharedPreferences)?

**Answer:**

```kotlin
// Preferences DataStore
val Context.dataStore by preferencesDataStore(name = "settings")

// Keys
val DARK_MODE = booleanPreferencesKey("dark_mode")
val USERNAME = stringPreferencesKey("username")

// Read
val darkMode: Flow<Boolean> = context.dataStore.data.map { prefs ->
    prefs[DARK_MODE] ?: false
}

// Write
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { prefs ->
        prefs[DARK_MODE] = enabled
    }
}

// Proto DataStore (type-safe, schema-based)
// Uses Protocol Buffers for type-safe structured data
```

**DataStore vs SharedPreferences:**

| Feature | SharedPreferences | DataStore |
|---------|------------------|-----------|
| Async | No (blocking on main thread) | Yes (Flow-based) |
| Type safety | No | Proto DataStore: Yes |
| Error handling | No | Yes |
| Transactional | No | Yes (atomic reads/writes) |

---

## Q8. What is Paging 3 library?

**Answer:**

```kotlin
// PagingSource
class UserPagingSource(private val api: ApiService) : PagingSource<Int, User>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        val page = params.key ?: 1
        return try {
            val response = api.getUsers(page, params.loadSize)
            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.users.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    override fun getRefreshKey(state: PagingState<Int, User>): Int? {
        return state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }
    }
}

// ViewModel
val users: Flow<PagingData<User>> = Pager(
    config = PagingConfig(pageSize = 20, enablePlaceholders = false),
    pagingSourceFactory = { UserPagingSource(api) }
).flow.cachedIn(viewModelScope)

// UI (Compose)
val lazyPagingItems = viewModel.users.collectAsLazyPagingItems()
LazyColumn {
    items(lazyPagingItems.itemCount) { index ->
        lazyPagingItems[index]?.let { UserItem(it) }
    }
    // Loading/error states
    when (lazyPagingItems.loadState.append) {
        is LoadState.Loading -> item { LoadingItem() }
        is LoadState.Error -> item { ErrorItem(retry = { lazyPagingItems.retry() }) }
        else -> {}
    }
}
```
