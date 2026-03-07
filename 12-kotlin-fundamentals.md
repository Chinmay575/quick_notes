# 12 — Kotlin Fundamentals (Android)

---

## Q1. What are the key features of Kotlin?

**Answer:**
- **Null safety** — `String` vs `String?`
- **Extension functions** — add methods to existing classes
- **Coroutines** — async programming without callbacks
- **Data classes** — auto-generated equals, hashCode, copy, toString
- **Sealed classes** — restricted class hierarchies
- **Smart casts** — automatic type casting after checks
- **Higher-order functions** — functions as parameters/return values
- **Companion objects** — static-like members
- **Delegation** — `by` keyword for delegation pattern
- **100% Java interop**

---

## Q2. Explain null safety in Kotlin.

**Answer:**

```kotlin
var name: String = "Alice"    // non-nullable
var name2: String? = null     // nullable

// Safe call
name2?.length                 // returns null if name2 is null

// Elvis operator
val len = name2?.length ?: 0  // default value if null

// Not-null assertion
name2!!.length                // throws NPE if null

// Safe cast
val x: String? = value as? String  // null if cast fails

// let (scoped null check)
name2?.let { println("Name is $it") }

// lateinit (initialized later, must be non-null)
lateinit var adapter: RecyclerView.Adapter<*>

// lazy (initialized on first access)
val db: Database by lazy { Database.create() }
```

---

## Q3. Explain Kotlin Coroutines.

**Answer:**

```kotlin
// Basic coroutine
viewModelScope.launch {
    val users = repository.getUsers()  // suspending function
    _uiState.value = UiState.Success(users)
}

// Suspending function
suspend fun getUsers(): List<User> {
    return withContext(Dispatchers.IO) {
        api.fetchUsers()
    }
}

// Dispatchers
Dispatchers.Main    // UI thread
Dispatchers.IO      // network/disk operations
Dispatchers.Default // CPU-intensive work

// async/await (parallel)
coroutineScope {
    val user = async { api.getUser(id) }
    val posts = async { api.getPosts(id) }
    val result = Pair(user.await(), posts.await()) // parallel execution
}

// Flow (cold stream, similar to Dart Streams)
fun getUsers(): Flow<List<User>> = flow {
    while (true) {
        emit(api.fetchUsers())
        delay(30_000) // refresh every 30s
    }
}.flowOn(Dispatchers.IO)

// Collect
viewModelScope.launch {
    repository.getUsers().collect { users ->
        _uiState.value = UiState.Success(users)
    }
}

// StateFlow and SharedFlow
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()
```

---

## Q4. What are data classes, sealed classes, and enum classes?

**Answer:**

```kotlin
// Data class — auto-generates equals, hashCode, copy, toString, destructuring
data class User(val id: Int, val name: String, val email: String)

val user = User(1, "Alice", "alice@test.com")
val copy = user.copy(name = "Bob")
val (id, name, email) = user  // destructuring

// Sealed class — restricted hierarchy, exhaustive when
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

when (result) {
    is Result.Success -> showData(result.data)
    is Result.Error -> showError(result.message)
    is Result.Loading -> showLoading()
    // No else needed — compiler verifies all cases
}

// Sealed interface (Kotlin 1.5+)
sealed interface UiEvent
data class Click(val id: String) : UiEvent
data class Scroll(val position: Int) : UiEvent

// Enum class
enum class Direction(val degrees: Int) {
    NORTH(0), SOUTH(180), EAST(90), WEST(270);

    fun opposite(): Direction = when (this) {
        NORTH -> SOUTH; SOUTH -> NORTH
        EAST -> WEST; WEST -> EAST
    }
}
```

---

## Q5. Explain extension functions and properties.

**Answer:**

```kotlin
// Extension function
fun String.isValidEmail(): Boolean {
    return android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()
}
"test@test.com".isValidEmail() // true

// Extension property
val String.wordCount: Int
    get() = this.split("\\s+".toRegex()).size
"Hello World".wordCount // 2

// Extension on nullable type
fun String?.orEmpty(): String = this ?: ""

// Scope functions (extensions on Any)
user?.let { print(it.name) }      // execute if non-null, returns result
user.also { log(it) }             // side effect, returns same object
user.apply { name = "Bob" }       // configure object, returns same object
user.run { "$name ($email)" }     // transform, returns result
with(user) { "$name ($email)" }   // same as run but not extension
```

---

## Q6. What are higher-order functions and lambdas?

**Answer:**

```kotlin
// Higher-order function (takes function as parameter)
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) result.add(item)
    }
    return result
}

val adults = users.customFilter { it.age >= 18 }

// Function types
val onClick: () -> Unit = { println("clicked") }
val transform: (String) -> Int = { it.length }
val combine: (Int, Int) -> Int = { a, b -> a + b }

// Inline functions (avoid lambda overhead)
inline fun <T> measureTime(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("Took ${System.currentTimeMillis() - start}ms")
    return result
}
```

---

## Q7. Explain Kotlin delegation.

**Answer:**

```kotlin
// Class delegation
interface Printer { fun print(msg: String) }
class ConsolePrinter : Printer {
    override fun print(msg: String) = println(msg)
}
class App(printer: Printer) : Printer by printer // delegates all Printer methods

// Property delegation
class User {
    var name: String by Delegates.observable("") { _, old, new ->
        println("Changed from $old to $new")
    }

    val lazyValue: String by lazy { expensiveComputation() }

    var preference: String by SharedPreferencesDelegate("pref_key")
}

// Custom delegate
class SharedPreferencesDelegate(private val key: String) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return prefs.getString(key, "") ?: ""
    }
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        prefs.edit().putString(key, value).apply()
    }
}
```

---

## Q8. What is the difference between `object`, `companion object`, and `class`?

**Answer:**

```kotlin
// object — Singleton
object AppConfig {
    val apiUrl = "https://api.example.com"
    fun init() { /* ... */ }
}
AppConfig.apiUrl // direct access

// companion object — static-like members inside a class
class User private constructor(val name: String) {
    companion object {
        fun create(name: String): User = User(name) // factory method
        const val MAX_NAME_LENGTH = 50
    }
}
User.create("Alice")
User.MAX_NAME_LENGTH

// Anonymous object
val listener = object : View.OnClickListener {
    override fun onClick(v: View?) { /* ... */ }
}
```

---

## Q9. Explain Kotlin Generics and variance.

**Answer:**

```kotlin
// Generic class
class Box<T>(val value: T)

// Bounded generics
fun <T : Comparable<T>> sort(list: List<T>) { /* ... */ }

// Covariance (out) — producer, read-only
interface Source<out T> {
    fun next(): T  // can return T
    // fun add(item: T) // ❌ cannot accept T
}
val strings: Source<String> = /* ... */
val objects: Source<Any> = strings // OK — String is subtype of Any

// Contravariance (in) — consumer, write-only
interface Sink<in T> {
    fun add(item: T)  // can accept T
    // fun next(): T   // ❌ cannot return T
}
val objects: Sink<Any> = /* ... */
val strings: Sink<String> = objects // OK

// Star projection
fun printAll(list: List<*>) { // unknown type, read-only as Any?
    list.forEach { println(it) }
}

// Reified type parameters (inline functions only)
inline fun <reified T> isType(value: Any): Boolean = value is T
```

---

## Q10. What are Kotlin Multiplatform (KMP) basics?

**Answer:**
KMP lets you share Kotlin code across Android, iOS, web, desktop.

```kotlin
// Shared module — common code
// commonMain
expect fun platformName(): String

class Greeting {
    fun greet(): String = "Hello from ${platformName()}"
}

// androidMain
actual fun platformName(): String = "Android"

// iosMain
actual fun platformName(): String = "iOS"
```

**What you can share:** Business logic, networking, data models, database, validation.
**What stays platform-specific:** UI, platform APIs, hardware access.
