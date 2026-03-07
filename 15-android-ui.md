# 15 — Android UI (XML & Jetpack Compose)

---

## Q1. Explain RecyclerView and its components.

**Answer:**

```kotlin
// Adapter with ListAdapter (DiffUtil)
class UserAdapter : ListAdapter<User, UserAdapter.ViewHolder>(UserDiffCallback()) {

    class ViewHolder(private val binding: ItemUserBinding) : RecyclerView.ViewHolder(binding.root) {
        fun bind(user: User) {
            binding.nameText.text = user.name
            binding.emailText.text = user.email
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return ViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(old: User, new: User) = old.id == new.id
    override fun areContentsTheSame(old: User, new: User) = old == new
}

// Setup
recyclerView.apply {
    adapter = userAdapter
    layoutManager = LinearLayoutManager(context)
    // or GridLayoutManager(context, 2)
    // or StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL)
    setHasFixedSize(true)
    addItemDecoration(DividerItemDecoration(context, DividerItemDecoration.VERTICAL))
}
```

---

## Q2. What is ViewBinding and DataBinding?

**Answer:**

```kotlin
// ViewBinding (recommended — simpler, type-safe)
// build.gradle: buildFeatures { viewBinding true }
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        binding.nameText.text = "Hello"
    }
}

// DataBinding (two-way binding with expressions in XML)
// build.gradle: buildFeatures { dataBinding true }
// XML:
// <layout>
//   <data><variable name="user" type="com.app.User" /></data>
//   <TextView android:text="@{user.name}" />
//   <EditText android:text="@={viewModel.query}" /> <!-- two-way -->
// </layout>
```

---

## Q3. Explain Jetpack Compose fundamentals.

**Answer:**

```kotlin
// Composable function
@Composable
fun UserCard(user: User, onClick: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp)
            .clickable { onClick() }
    ) {
        Row(modifier = Modifier.padding(16.dp)) {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = "Avatar",
                modifier = Modifier.size(48.dp).clip(CircleShape)
            )
            Spacer(modifier = Modifier.width(16.dp))
            Column {
                Text(user.name, style = MaterialTheme.typography.titleMedium)
                Text(user.email, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}

// State management in Compose
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    // remember survives recomposition but NOT configuration change

    var count2 by rememberSaveable { mutableStateOf(0) }
    // rememberSaveable survives configuration change

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

---

## Q4. Explain Compose state hoisting and unidirectional data flow.

**Answer:**

```kotlin
// ❌ Stateful component (hard to test/reuse)
@Composable
fun SearchBar() {
    var query by remember { mutableStateOf("") }
    TextField(value = query, onValueChange = { query = it })
}

// ✅ Stateless component (state hoisted to parent)
@Composable
fun SearchBar(query: String, onQueryChange: (String) -> Unit) {
    TextField(value = query, onValueChange = onQueryChange)
}

// Parent owns the state
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column {
        SearchBar(query = query, onQueryChange = viewModel::onQueryChange)
        ResultsList(results = results)
    }
}
```

**Unidirectional data flow:** Events go UP, state goes DOWN.

---

## Q5. What is recomposition in Compose? How do you optimize it?

**Answer:**

Recomposition = Compose re-executing composable functions when state changes.

```kotlin
// Compose skips recomposition of functions with unchanged parameters
// For this to work, parameters must be "stable"

// ✅ Stable types (primitives, String, data classes with stable fields)
data class User(val id: Int, val name: String) // stable

// ❌ Unstable types
class User(var name: String) // mutable = unstable

// @Stable annotation
@Stable
class UiState(val items: List<Item>) // mark as stable manually

// Optimization techniques:
// 1. Use remember to avoid recomputation
val sorted = remember(items) { items.sortedBy { it.name } }

// 2. Use derivedStateOf for computed state
val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }

// 3. Lambda stability — use method references
Button(onClick = viewModel::onSubmit) // stable reference

// 4. Use key() to control identity
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserItem(user)
    }
}
```

---

## Q6. Explain Modifier in Compose.

**Answer:**

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .fillMaxWidth()           // sizing
        .padding(16.dp)           // spacing
        .background(Color.Blue)   // decoration
        .clip(RoundedCornerShape(8.dp))  // shape
        .clickable { onClick() }  // interaction
        .alpha(0.5f)              // visual effect
        .border(1.dp, Color.Gray) // border
        .shadow(4.dp)             // elevation
        .testTag("greeting")      // testing
)

// ORDER MATTERS!
// padding then background ≠ background then padding
Modifier.padding(16.dp).background(Color.Red)  // red background inside padding
Modifier.background(Color.Red).padding(16.dp)  // red background outside padding
```

---

## Q7. Explain Compose Navigation.

**Answer:**

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onNavigateToDetail = { id ->
                navController.navigate("detail/$id")
            })
        }

        composable(
            route = "detail/{userId}",
            arguments = listOf(navArgument("userId") { type = NavType.IntType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getInt("userId") ?: 0
            DetailScreen(userId = userId)
        }

        // Nested navigation
        navigation(startDestination = "login", route = "auth") {
            composable("login") { LoginScreen() }
            composable("register") { RegisterScreen() }
        }
    }
}
```

---

## Q8. What is `LazyColumn` and `LazyRow`?

**Answer:**

```kotlin
LazyColumn(
    modifier = Modifier.fillMaxSize(),
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    // Header
    item { Text("Users", style = MaterialTheme.typography.headlineMedium) }

    // List items with keys
    items(users, key = { it.id }) { user ->
        UserItem(user = user)
    }

    // Sticky headers
    stickyHeader { Text("Section A") }
    items(sectionA) { item -> ItemCard(item) }

    // Loading indicator
    if (isLoading) {
        item { CircularProgressIndicator() }
    }
}

// LazyVerticalGrid
LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 128.dp),
    contentPadding = PaddingValues(8.dp)
) {
    items(items) { item -> GridItem(item) }
}
```

---

## Q9. How do you handle side effects in Compose?

**Answer:**

```kotlin
// LaunchedEffect — runs coroutine when key changes
LaunchedEffect(userId) {
    viewModel.loadUser(userId) // runs once per userId change
}

// DisposableEffect — setup + cleanup
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> /* ... */ }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}

// SideEffect — runs after every successful recomposition
SideEffect {
    analytics.setScreen(screenName) // called on every recomposition
}

// rememberCoroutineScope — coroutine scope for event handlers
val scope = rememberCoroutineScope()
Button(onClick = { scope.launch { viewModel.save() } })

// produceState — convert non-Compose state to Compose state
val user by produceState<User?>(initialValue = null, userId) {
    value = repository.getUser(userId)
}

// snapshotFlow — convert Compose state to Flow
LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .distinctUntilChanged()
        .collect { index -> analytics.trackScroll(index) }
}
```

---

## Q10. What is `CompositionLocal`?

**Answer:**

```kotlin
// Define
val LocalUserSession = compositionLocalOf<UserSession> { error("No session provided") }

// Provide
CompositionLocalProvider(LocalUserSession provides userSession) {
    MyApp() // all descendants can access
}

// Access
@Composable
fun ProfileScreen() {
    val session = LocalUserSession.current
    Text("Welcome, ${session.user.name}")
}

// Built-in CompositionLocals
LocalContext.current          // Android Context
LocalLifecycleOwner.current   // LifecycleOwner
LocalDensity.current          // Density for dp/px conversion
LocalConfiguration.current    // Configuration (orientation, locale)
```
