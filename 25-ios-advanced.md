# 25 — iOS Advanced Topics

---

## Q1. Explain Swift Concurrency (async/await, actors).

**Answer:**

```swift
// Async function
func fetchUser(id: Int) async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Structured concurrency — async let (parallel)
func fetchDashboard() async throws -> Dashboard {
    async let user = fetchUser(id: 1)
    async let posts = fetchPosts(userId: 1)
    async let notifications = fetchNotifications()
    // All three run in parallel
    return try await Dashboard(user: user, posts: posts, notifications: notifications)
}

// TaskGroup (dynamic parallelism)
func fetchAllUsers(ids: [Int]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask { try await self.fetchUser(id: id) }
        }
        var users: [User] = []
        for try await user in group { users.append(user) }
        return users
    }
}

// Task (unstructured)
Task { await viewModel.loadData() }
Task.detached(priority: .background) { await heavyWork() }

// Actor — thread-safe state
actor UserCache {
    private var cache: [Int: User] = [:]

    func getUser(_ id: Int) -> User? { cache[id] }
    func setUser(_ user: User) { cache[user.id] = user }
}

let cache = UserCache()
await cache.setUser(user)      // must await actor methods
let user = await cache.getUser(1)

// @MainActor — runs on main thread
@MainActor
class UserViewModel: ObservableObject {
    @Published var users: [User] = []

    func loadUsers() async {
        let users = try? await repository.fetchUsers()
        self.users = users ?? [] // guaranteed main thread
    }
}

// Sendable — safe to pass across concurrency boundaries
struct User: Sendable { let name: String } // value types are Sendable
final class Config: Sendable { let apiUrl: String } // immutable class
```

---

## Q2. Explain Combine framework.

**Answer:**

```swift
import Combine

// Publishers and Subscribers
let publisher = URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .decode(type: [User].self, decoder: JSONDecoder())
    .receive(on: DispatchQueue.main)
    .sink(
        receiveCompletion: { completion in
            if case .failure(let error) = completion { print(error) }
        },
        receiveValue: { users in self.users = users }
    )

// Common operators
publisher
    .map { $0.name }           // transform values
    .filter { !$0.isEmpty }    // filter values
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main) // debounce
    .removeDuplicates()        // skip consecutive duplicates
    .flatMap { id in api.fetchUser(id) } // chain async operations
    .retry(3)                  // retry on failure
    .catch { _ in Just([]) }   // replace error with default
    .eraseToAnyPublisher()     // type erasure

// Subjects
let subject = PassthroughSubject<String, Never>() // no initial value
let currentValue = CurrentValueSubject<Int, Never>(0) // has initial value

subject.send("Hello")
currentValue.value = 42

// @Published (in ObservableObject)
class ViewModel: ObservableObject {
    @Published var searchText = ""
    @Published var results: [Item] = []

    init() {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .flatMap { query in self.search(query) }
            .assign(to: &$results)
    }
}

// Cancellation
var cancellables = Set<AnyCancellable>()
publisher.store(in: &cancellables)
// All subscriptions cancelled when cancellables is deallocated
```

---

## Q3. How do you implement Push Notifications?

**Answer:**

```swift
// 1. Request permission
import UserNotifications

func requestNotificationPermission() async -> Bool {
    let center = UNUserNotificationCenter.current()
    do {
        let granted = try await center.requestAuthorization(options: [.alert, .sound, .badge])
        if granted {
            await MainActor.run { UIApplication.shared.registerForRemoteNotifications() }
        }
        return granted
    } catch {
        return false
    }
}

// 2. Handle token
func application(_: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    // Send token to your server
    api.registerPushToken(token)
}

// 3. Handle received notification
class NotificationDelegate: NSObject, UNUserNotificationCenterDelegate {
    // Foreground
    func userNotificationCenter(_: UNUserNotificationCenter, willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        return [.banner, .sound, .badge]
    }

    // User tapped notification
    func userNotificationCenter(_: UNUserNotificationCenter, didReceive response: UNNotificationResponse) async {
        let userInfo = response.notification.request.content.userInfo
        handleDeepLink(from: userInfo)
    }
}

// 4. Local notifications
let content = UNMutableNotificationContent()
content.title = "Reminder"
content.body = "Don't forget to check your tasks"
content.sound = .default

let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 3600, repeats: false)
let request = UNNotificationRequest(identifier: "reminder", content: content, trigger: trigger)
try await UNUserNotificationCenter.current().add(request)
```

---

## Q4. Explain App Store submission process.

**Answer:**

1. **Prepare app** — Final testing, remove debug code, set version/build number
2. **Create App Store Connect record** — App name, description, screenshots, keywords
3. **Code signing** — Certificate + Provisioning Profile (distribution)
4. **Archive** — Xcode → Product → Archive
5. **Upload** — Xcode Organizer → Distribute App → App Store Connect
6. **TestFlight** — Internal/external testing
7. **Submit for review** — Fill metadata, submit
8. **App Review** — Apple reviews (1-3 days typically)
9. **Release** — Manual, automatic, or phased release

**Common rejection reasons:**
- Crashes / bugs
- Incomplete metadata / placeholder content
- Privacy policy missing
- Login credentials not provided for review
- Using private APIs
- In-app purchases not using Apple's payment system

---

## Q5. Explain iOS memory management and optimization.

**Answer:**

```swift
// 1. Avoid retain cycles
class VC: UIViewController {
    var closure: (() -> Void)?

    func setup() {
        // ❌ Retain cycle
        closure = { self.doSomething() }
        // ✅ Fixed
        closure = { [weak self] in self?.doSomething() }
    }
}

// 2. Use autoreleasepool for loops creating many objects
for i in 0..<100000 {
    autoreleasepool {
        let image = processImage(at: i) // temporary objects released each iteration
    }
}

// 3. Lazy loading
class DataManager {
    lazy var heavyObject = createHeavyObject() // only created when first accessed
}

// 4. Image optimization
let url = URL(string: "https://example.com/large.jpg")!
// Use thumbnail for lists
let request = URLRequest(url: url)
let (data, _) = try await URLSession.shared.data(for: request)
let image = UIImage(data: data)
let thumbnail = image?.preparingThumbnail(of: CGSize(width: 100, height: 100))

// 5. Respond to memory warnings
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    imageCache.removeAllObjects()
}
```

**Tools:** Instruments → Leaks, Allocations, Zombies. Xcode Memory Graph Debugger.

---

## Q6. What is App Thinning?

**Answer:**

| Feature | Purpose |
|---------|---------|
| **Slicing** | App Store creates variants for each device (removes unused assets) |
| **Bitcode** | Apple re-optimizes binary for specific architectures (deprecated in Xcode 14) |
| **On-Demand Resources** | Download resources only when needed (game levels, tutorials) |

```swift
// On-demand resources
let request = NSBundleResourceRequest(tags: ["level-5"])
try await request.beginAccessingResources()
// Use resources...
request.endAccessingResources()
```

---

## Q7. Explain Widgets and App Extensions.

**Answer:**

```swift
// Widget
struct MyWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "MyWidget", provider: MyTimelineProvider()) { entry in
            MyWidgetView(entry: entry)
        }
        .configurationDisplayName("My Widget")
        .description("Shows recent data")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}

struct MyTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> MyEntry { MyEntry(date: .now) }

    func getSnapshot(in context: Context, completion: @escaping (MyEntry) -> Void) {
        completion(MyEntry(date: .now))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<MyEntry>) -> Void) {
        let entries = [MyEntry(date: .now)]
        let timeline = Timeline(entries: entries, policy: .after(.now.addingTimeInterval(3600)))
        completion(timeline)
    }
}

// Common extensions
// Share Extension — share content from other apps
// Today Widget — widget on home screen
// Notification Content Extension — custom notification UI
// Intents Extension — Siri shortcuts
// App Clip — lightweight app experience
```
