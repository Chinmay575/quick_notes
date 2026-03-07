# 16 — Android Architecture

---

## Q1. Explain MVVM pattern in Android.

**Answer:**

```kotlin
// Model (data layer)
data class User(val id: Int, val name: String, val email: String)

class UserRepository @Inject constructor(
    private val api: ApiService,
    private val dao: UserDao
) {
    fun getUsers(): Flow<List<User>> = dao.getAllUsers()

    suspend fun refreshUsers() {
        val users = api.fetchUsers()
        dao.insertAll(users)
    }
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    val users: StateFlow<List<User>> = repository.getUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            repository.refreshUsers()
            _isRefreshing.value = false
        }
    }
}

// View (Compose)
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    SwipeRefresh(state = rememberSwipeRefreshState(isRefreshing), onRefresh = viewModel::refresh) {
        LazyColumn { items(users) { user -> UserItem(user) } }
    }
}
```

---

## Q2. Explain MVI (Model-View-Intent) pattern.

**Answer:**

```kotlin
// State (immutable)
data class UserState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

// Intent/Event
sealed class UserIntent {
    object LoadUsers : UserIntent()
    data class SearchUsers(val query: String) : UserIntent()
    data class DeleteUser(val id: Int) : UserIntent()
}

// Side effects (one-time events)
sealed class UserEffect {
    data class ShowToast(val message: String) : UserEffect()
    object NavigateBack : UserEffect()
}

// ViewModel
class UserViewModel : ViewModel() {
    private val _state = MutableStateFlow(UserState())
    val state: StateFlow<UserState> = _state.asStateFlow()

    private val _effect = Channel<UserEffect>()
    val effect: Flow<UserEffect> = _effect.receiveAsFlow()

    fun handleIntent(intent: UserIntent) {
        when (intent) {
            is UserIntent.LoadUsers -> loadUsers()
            is UserIntent.SearchUsers -> searchUsers(intent.query)
            is UserIntent.DeleteUser -> deleteUser(intent.id)
        }
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val users = repository.getUsers()
                _state.update { it.copy(users = users, isLoading = false) }
            } catch (e: Exception) {
                _state.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}
```

**MVI vs MVVM:**
- MVI: Single state object, unidirectional, events via sealed class
- MVVM: Multiple observable fields, bidirectional possible

---

## Q3. Explain Hilt dependency injection.

**Answer:**

```kotlin
// 1. Application
@HiltAndroidApp
class MyApp : Application()

// 2. Module (provide dependencies)
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com")
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService =
        retrofit.create(ApiService::class.java)

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
}

// 3. Repository
@Singleton
class UserRepository @Inject constructor(
    private val api: ApiService,
    private val db: AppDatabase
) { /* ... */ }

// 4. ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()

// 5. Activity/Fragment
@AndroidEntryPoint
class MainActivity : AppCompatActivity()

@AndroidEntryPoint
class UserFragment : Fragment()
```

**Hilt scopes:**

| Component | Scope | Lifetime |
|-----------|-------|----------|
| SingletonComponent | @Singleton | Application |
| ActivityRetainedComponent | @ActivityRetainedScoped | ViewModel |
| ViewModelComponent | @ViewModelScoped | ViewModel |
| ActivityComponent | @ActivityScoped | Activity |
| FragmentComponent | @FragmentScoped | Fragment |

---

## Q4. What is the Repository pattern in Android?

**Answer:**

```kotlin
class UserRepository @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val networkChecker: NetworkChecker
) {
    fun getUsers(): Flow<Resource<List<User>>> = flow {
        emit(Resource.Loading())

        // Emit cached data first
        val cachedUsers = localDataSource.getUsers().first()
        if (cachedUsers.isNotEmpty()) emit(Resource.Success(cachedUsers))

        // Fetch from network
        if (networkChecker.isOnline()) {
            try {
                val remoteUsers = remoteDataSource.fetchUsers()
                localDataSource.insertAll(remoteUsers)
                emitAll(localDataSource.getUsers().map { Resource.Success(it) })
            } catch (e: Exception) {
                emit(Resource.Error(e.message ?: "Unknown", cachedUsers))
            }
        }
    }
}

sealed class Resource<T>(val data: T? = null, val message: String? = null) {
    class Success<T>(data: T) : Resource<T>(data)
    class Error<T>(message: String, data: T? = null) : Resource<T>(data, message)
    class Loading<T>(data: T? = null) : Resource<T>(data)
}
```

---

## Q5. Explain Clean Architecture in Android.

**Answer:**

```
app/
├── di/                     # Hilt modules
├── data/
│   ├── remote/            # Retrofit API interfaces, DTOs
│   ├── local/             # Room DAOs, entities
│   ├── mapper/            # DTO → Domain mapping
│   └── repository/        # Repository implementations
├── domain/
│   ├── model/             # Business models
│   ├── repository/        # Repository interfaces
│   └── usecase/           # Business logic
└── presentation/
    ├── ui/                # Composables/Fragments
    └── viewmodel/         # ViewModels
```

```kotlin
// Use Case
class GetUsersUseCase @Inject constructor(private val repository: UserRepository) {
    operator fun invoke(): Flow<Resource<List<User>>> = repository.getUsers()
}

// ViewModel uses UseCase, not Repository directly
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUsers: GetUsersUseCase
) : ViewModel() {
    val users = getUsers().stateIn(viewModelScope, SharingStarted.Lazily, Resource.Loading())
}
```

---

## Q6. What is Koin vs Hilt?

**Answer:**

| Feature | Hilt | Koin |
|---------|------|------|
| Type | Compile-time DI (Dagger-based) | Runtime service locator |
| Errors | Compile-time | Runtime |
| Performance | Faster (generated code) | Slightly slower (reflection) |
| Boilerplate | More annotations | Less code |
| Learning curve | Steeper | Easier |
| Testing | Built-in test rules | Simple module overrides |
| Multi-module | Good support | Good support |

```kotlin
// Koin
val appModule = module {
    single { Retrofit.Builder().baseUrl("...").build() }
    single { get<Retrofit>().create(ApiService::class.java) }
    factory { UserRepository(get()) }
    viewModel { UserViewModel(get()) }
}

startKoin { modules(appModule) }

// In ViewModel
class UserViewModel(private val repo: UserRepository) : ViewModel()

// In Activity
val viewModel: UserViewModel by viewModel()
```
