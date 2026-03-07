# 23 — iOS Data & Networking

---

## Q1. Explain URLSession.

**Answer:**

```swift
// GET request
func fetchUsers() async throws -> [User] {
    let url = URL(string: "https://api.example.com/users")!
    var request = URLRequest(url: url)
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    request.cachePolicy = .returnCacheDataElseLoad

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw NetworkError.invalidResponse
    }

    return try JSONDecoder().decode([User].self, from: data)
}

// POST request
func createUser(_ user: User) async throws -> User {
    var request = URLRequest(url: URL(string: "https://api.example.com/users")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(user)

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(User.self, from: data)
}

// Upload with progress
func upload(data: Data) async throws {
    let (_, response) = try await URLSession.shared.upload(
        for: request,
        from: data,
        delegate: self // URLSessionTaskDelegate for progress
    )
}

// Download
let (url, _) = try await URLSession.shared.download(from: downloadURL)
try FileManager.default.moveItem(at: url, to: destinationURL)

// Custom session configuration
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 30
config.waitsForConnectivity = true
config.httpAdditionalHeaders = ["Accept": "application/json"]
let session = URLSession(configuration: config)
```

---

## Q2. Explain Core Data.

**Answer:**

```swift
// Core Data Stack
class PersistenceController {
    static let shared = PersistenceController()
    let container: NSPersistentContainer

    init() {
        container = NSPersistentContainer(name: "MyApp")
        container.loadPersistentStores { _, error in
            if let error = error { fatalError("Core Data error: \(error)") }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
}

// Entity (defined in .xcdatamodeld or code)
@objc(UserEntity)
class UserEntity: NSManagedObject {
    @NSManaged var id: UUID
    @NSManaged var name: String
    @NSManaged var email: String
    @NSManaged var posts: NSSet?  // relationship
}

// CRUD Operations
let context = PersistenceController.shared.container.viewContext

// Create
let user = UserEntity(context: context)
user.id = UUID()
user.name = "Alice"
user.email = "alice@test.com"
try context.save()

// Read
let request: NSFetchRequest<UserEntity> = UserEntity.fetchRequest()
request.predicate = NSPredicate(format: "name CONTAINS[cd] %@", "Alice")
request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
request.fetchLimit = 20
let users = try context.fetch(request)

// Update
user.name = "Alice Updated"
try context.save()

// Delete
context.delete(user)
try context.save()

// SwiftUI integration
@FetchRequest(
    sortDescriptors: [SortDescriptor(\.name)],
    predicate: NSPredicate(format: "isActive == true")
)
private var users: FetchedResults<UserEntity>
```

---

## Q3. What is SwiftData (iOS 17+)?

**Answer:**

```swift
// Model (replaces Core Data entity)
@Model
class User {
    var name: String
    var email: String
    var createdAt: Date
    @Relationship(deleteRule: .cascade) var posts: [Post]

    init(name: String, email: String) {
        self.name = name
        self.email = email
        self.createdAt = Date()
        self.posts = []
    }
}

// Setup
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
            .modelContainer(for: [User.self, Post.self])
    }
}

// CRUD in SwiftUI
struct UserListView: View {
    @Environment(\.modelContext) private var context
    @Query(sort: \User.name) private var users: [User]

    var body: some View {
        List(users) { user in Text(user.name) }
    }

    func addUser() {
        let user = User(name: "Alice", email: "alice@test.com")
        context.insert(user)
        // Auto-saves!
    }

    func deleteUser(_ user: User) {
        context.delete(user)
    }
}

// Filtered query
@Query(filter: #Predicate<User> { $0.name.contains("Alice") })
private var filteredUsers: [User]
```

---

## Q4. Explain Keychain for secure storage.

**Answer:**

```swift
class KeychainManager {
    static func save(key: String, data: Data) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        SecItemDelete(query as CFDictionary) // remove existing
        let status = SecItemAdd(query as CFDictionary, nil)
        return status == errSecSuccess
    }

    static func load(key: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        return result as? Data
    }

    static func delete(key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        SecItemDelete(query as CFDictionary)
    }
}

// Usage
let token = "eyJhbGci..."
KeychainManager.save(key: "auth_token", data: token.data(using: .utf8)!)
let savedToken = String(data: KeychainManager.load(key: "auth_token")!, encoding: .utf8)
```

---

## Q5. Explain UserDefaults.

**Answer:**

```swift
// Write
UserDefaults.standard.set("Alice", forKey: "username")
UserDefaults.standard.set(true, forKey: "darkMode")
UserDefaults.standard.set(42, forKey: "loginCount")
UserDefaults.standard.set(["a", "b", "c"], forKey: "tags")

// Read
let name = UserDefaults.standard.string(forKey: "username") // Optional
let darkMode = UserDefaults.standard.bool(forKey: "darkMode") // false if not set
let count = UserDefaults.standard.integer(forKey: "loginCount") // 0 if not set

// Register defaults
UserDefaults.standard.register(defaults: [
    "darkMode": false,
    "fontSize": 14,
    "language": "en"
])

// Observe changes
UserDefaults.standard.addObserver(self, forKeyPath: "darkMode", options: .new, context: nil)

// SwiftUI @AppStorage
@AppStorage("darkMode") var isDarkMode = false
@AppStorage("username") var username = "Guest"
```

**Do NOT store:** Sensitive data (tokens, passwords), large data, images.

---

## Q6. Explain Alamofire.

**Answer:**

```swift
// GET
AF.request("https://api.example.com/users")
    .validate()
    .responseDecodable(of: [User].self) { response in
        switch response.result {
        case .success(let users): self.users = users
        case .failure(let error): print(error)
        }
    }

// POST
AF.request("https://api.example.com/users",
           method: .post,
           parameters: ["name": "Alice", "email": "alice@test.com"],
           encoder: JSONParameterEncoder.default)
    .validate()
    .responseDecodable(of: User.self) { response in /* ... */ }

// Upload
AF.upload(multipartFormData: { formData in
    formData.append(imageData, withName: "photo", fileName: "photo.jpg", mimeType: "image/jpeg")
    formData.append("Alice".data(using: .utf8)!, withName: "name")
}, to: "https://api.example.com/upload")
.uploadProgress { progress in
    print("Upload Progress: \(progress.fractionCompleted)")
}

// Interceptor (auth token)
class AuthInterceptor: RequestInterceptor {
    func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
        var request = urlRequest
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        completion(.success(request))
    }

    func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
        if let response = request.task?.response as? HTTPURLResponse, response.statusCode == 401 {
            refreshToken { newToken in
                completion(.retry)
            }
        } else {
            completion(.doNotRetry)
        }
    }
}
```
