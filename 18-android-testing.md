# 18 — Android Testing

---

## Q1. Explain Android testing pyramid.

**Answer:**

| Layer | Tools | Speed | What it tests |
|-------|-------|-------|---------------|
| **Unit tests** | JUnit, Mockito/MockK, Turbine | Fast | ViewModels, repositories, use cases |
| **Integration tests** | Robolectric, Room testing | Medium | Component interactions |
| **UI tests** | Espresso, Compose Testing, UI Automator | Slow | User flows, screen rendering |

---

## Q2. How do you test ViewModels?

**Answer:**

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule() // replaces Dispatchers.Main

    private lateinit var viewModel: UserViewModel
    private val repository: UserRepository = mockk()

    @Before
    fun setup() {
        coEvery { repository.getUsers() } returns flowOf(listOf(testUser))
        viewModel = UserViewModel(repository)
    }

    @Test
    fun `loadUsers emits success state`() = runTest {
        viewModel.uiState.test { // Turbine
            assertEquals(UiState.Loading, awaitItem())
            assertEquals(UiState.Success(listOf(testUser)), awaitItem())
        }
    }

    @Test
    fun `loadUsers emits error on failure`() = runTest {
        coEvery { repository.getUsers() } throws IOException("Network error")
        viewModel = UserViewModel(repository)

        viewModel.uiState.test {
            assertEquals(UiState.Loading, awaitItem())
            val error = awaitItem() as UiState.Error
            assertEquals("Network error", error.message)
        }
    }
}

// MainDispatcherRule
class MainDispatcherRule : TestWatcher() {
    val testDispatcher = UnconfinedTestDispatcher()
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## Q3. How do you test Room DAOs?

**Answer:**

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var database: AppDatabase
    private lateinit var dao: UserDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        dao = database.userDao()
    }

    @After
    fun teardown() { database.close() }

    @Test
    fun insertAndReadUser() = runTest {
        val user = UserEntity(name = "Alice", email = "alice@test.com")
        dao.insert(user)

        val users = dao.getAllUsers().first()
        assertEquals(1, users.size)
        assertEquals("Alice", users[0].name)
    }
}
```

---

## Q4. How do you test Jetpack Compose UI?

**Answer:**

```kotlin
class UserScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun displaysUserList() {
        composeTestRule.setContent {
            UserScreen(users = listOf(User(1, "Alice"), User(2, "Bob")))
        }

        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
        composeTestRule.onNodeWithText("Bob").assertIsDisplayed()
    }

    @Test
    fun clickingItemNavigates() {
        var clickedId: Int? = null

        composeTestRule.setContent {
            UserScreen(
                users = listOf(User(1, "Alice")),
                onUserClick = { clickedId = it }
            )
        }

        composeTestRule.onNodeWithText("Alice").performClick()
        assertEquals(1, clickedId)
    }

    @Test
    fun searchFiltersResults() {
        composeTestRule.setContent { SearchScreen() }

        composeTestRule.onNodeWithTag("search_field").performTextInput("Alice")
        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
        composeTestRule.onNodeWithText("Bob").assertDoesNotExist()
    }
}
```

---

## Q5. How do you test with Espresso?

**Answer:**

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginActivityTest {
    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)

    @Test
    fun validLoginNavigatesToHome() {
        // Type in fields
        onView(withId(R.id.emailField)).perform(typeText("test@test.com"), closeSoftKeyboard())
        onView(withId(R.id.passwordField)).perform(typeText("password123"), closeSoftKeyboard())

        // Click login
        onView(withId(R.id.loginButton)).perform(click())

        // Verify navigation
        onView(withText("Welcome")).check(matches(isDisplayed()))
    }

    @Test
    fun emptyEmailShowsError() {
        onView(withId(R.id.loginButton)).perform(click())
        onView(withText("Email is required")).check(matches(isDisplayed()))
    }

    @Test
    fun recyclerViewHasItems() {
        onView(withId(R.id.recyclerView))
            .perform(RecyclerViewActions.scrollToPosition<RecyclerView.ViewHolder>(10))

        onView(withId(R.id.recyclerView))
            .check(matches(hasMinimumChildCount(1)))
    }
}
```

---

## Q6. How do you use MockK for Kotlin?

**Answer:**

```kotlin
// MockK — Kotlin-first mocking library
val repository = mockk<UserRepository>()

// Stub
every { repository.getUser(1) } returns User(1, "Alice")
coEvery { repository.fetchUsers() } returns listOf(testUser) // suspending
every { repository.users } returns flowOf(listOf(testUser))   // Flow

// Verify
verify { repository.getUser(1) }
verify(exactly = 2) { repository.getUser(any()) }
coVerify { repository.fetchUsers() }
confirmVerified(repository)

// Relaxed mock (returns default values for unstubbed calls)
val relaxedMock = mockk<UserRepository>(relaxed = true)

// Spy (partial mock)
val spy = spyk(RealRepository())
every { spy.getUser(1) } returns mockUser // override specific methods

// Capture arguments
val slot = slot<User>()
every { repository.save(capture(slot)) } returns Unit
repository.save(User(1, "Alice"))
assertEquals("Alice", slot.captured.name)
```
