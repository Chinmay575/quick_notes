# 20 — Swift Fundamentals

---

## Q1. Explain Swift's type system — value types vs reference types.

**Answer:**

| Feature | Value Type (struct, enum) | Reference Type (class) |
|---------|--------------------------|----------------------|
| Copy behavior | Copied on assignment | Shared reference |
| Memory | Stack (usually) | Heap |
| Inheritance | No | Yes |
| Deinit | No | Yes |
| Mutability | `mutating` keyword needed | Mutable by default |
| Thread safety | Inherently safer (copy) | Requires synchronization |

```swift
// Struct (value type) — preferred in Swift
struct Point {
    var x: Double
    var y: Double

    mutating func moveBy(dx: Double, dy: Double) {
        x += dx
        y += dy
    }
}

var p1 = Point(x: 0, y: 0)
var p2 = p1  // COPY
p2.x = 10
print(p1.x)  // 0 (unchanged)

// Class (reference type)
class Person {
    var name: String
    init(name: String) { self.name = name }
    deinit { print("Person deinitialized") }
}

let person1 = Person(name: "Alice")
let person2 = person1  // SHARED reference
person2.name = "Bob"
print(person1.name)  // "Bob" (changed!)
```

**Rule of thumb:** Use structs by default. Use classes when you need identity, inheritance, or Objective-C interop.

---

## Q2. Explain Optionals in Swift.

**Answer:**

```swift
var name: String? = nil  // Optional — may or may not have a value

// Unwrapping
// 1. Optional binding (if let)
if let name = name {
    print("Name is \(name)")
}

// 2. Guard let (early exit)
guard let name = name else {
    print("No name")
    return
}
print(name) // safely unwrapped

// 3. Optional chaining
let count = name?.count  // nil if name is nil

// 4. Nil coalescing
let displayName = name ?? "Unknown"

// 5. Force unwrap (dangerous!)
let forcedName = name!  // crashes if nil

// 6. Implicitly unwrapped optional
var connection: Connection!  // assumed non-nil after initialization

// Optional map/flatMap
let number: Int? = 42
let string = number.map { String($0) }  // Optional("42")
let parsed: Int? = "42".flatMap { Int(String($0)) }
```

---

## Q3. Explain protocols in Swift.

**Answer:**

```swift
protocol Drawable {
    var color: UIColor { get set }
    func draw()
}

// Protocol with default implementation (protocol extension)
extension Drawable {
    func draw() {
        print("Drawing with \(color)")
    }
}

// Protocol with associated type
protocol Repository {
    associatedtype Entity
    func getAll() async throws -> [Entity]
    func getById(_ id: String) async throws -> Entity?
    func save(_ entity: Entity) async throws
}

class UserRepository: Repository {
    typealias Entity = User
    func getAll() async throws -> [User] { /* ... */ }
    func getById(_ id: String) async throws -> User? { /* ... */ }
    func save(_ entity: User) async throws { /* ... */ }
}

// Protocol composition
func processItem(_ item: Drawable & Codable) { /* ... */ }

// Protocol as type constraint
func findFirst<T: Equatable>(in array: [T], matching value: T) -> T? {
    return array.first { $0 == value }
}
```

---

## Q4. Explain ARC (Automatic Reference Counting).

**Answer:**

```swift
class Person {
    let name: String
    var apartment: Apartment?
    init(name: String) { self.name = name }
    deinit { print("\(name) deinitialized") }
}

class Apartment {
    let unit: String
    weak var tenant: Person?  // WEAK — doesn't increment RC
    init(unit: String) { self.unit = unit }
}

// Strong reference cycle (memory leak)
class A { var b: B? }
class B { var a: A? }  // ❌ both hold strong references → leak

// Fix with weak or unowned
class B { weak var a: A? }      // weak — optional, can become nil
class B { unowned var a: A }    // unowned — non-optional, must outlive B

// Closures and capture lists
class ViewController {
    var name = "Main"

    func setupCallback() {
        // ❌ Strong capture → retain cycle
        networkManager.onComplete = {
            self.updateUI()  // self is strongly captured
        }

        // ✅ Weak capture
        networkManager.onComplete = { [weak self] in
            guard let self = self else { return }
            self.updateUI()
        }

        // ✅ Unowned (when you know self will exist)
        networkManager.onComplete = { [unowned self] in
            self.updateUI()
        }
    }
}
```

| Keyword | Reference count | Becomes nil? | Use when |
|---------|----------------|-------------|----------|
| strong | +1 | No | Default, ownership |
| weak | 0 | Yes (set to nil) | Delegates, parent refs |
| unowned | 0 | No (crash if nil) | Same lifetime guaranteed |

---

## Q5. Explain generics in Swift.

**Answer:**

```swift
// Generic function
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a; a = b; b = temp
}

// Generic type
struct Stack<Element> {
    private var items: [Element] = []
    mutating func push(_ item: Element) { items.append(item) }
    mutating func pop() -> Element? { items.popLast() }
    var peek: Element? { items.last }
}

// Type constraints
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    return array.firstIndex(of: value)
}

// Where clause
func allItemsMatch<C1: Container, C2: Container>(_ a: C1, _ b: C2) -> Bool
    where C1.Item == C2.Item, C1.Item: Equatable { /* ... */ }

// Opaque types (some)
func makeShape() -> some Shape {
    return Circle(radius: 5) // caller doesn't know concrete type
}
```

---

## Q6. Explain closures in Swift.

**Answer:**

```swift
// Full syntax
let add: (Int, Int) -> Int = { (a: Int, b: Int) -> Int in
    return a + b
}

// Shorthand
let add = { $0 + $1 }

// Trailing closure
numbers.sorted { $0 > $1 }

// @escaping — closure outlives the function (async, stored)
func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, error in
        if let data = data {
            completion(.success(data))
        } else {
            completion(.failure(error!))
        }
    }.resume()
}

// @autoclosure — wraps expression in closure automatically
func assert(_ condition: @autoclosure () -> Bool, _ message: String) {
    if !condition() { print(message) }
}
assert(x > 0, "x must be positive") // x > 0 is auto-wrapped
```

---

## Q7. What are property wrappers?

**Answer:**

```swift
// Built-in: @State, @Binding, @Published, @ObservedObject, etc.

// Custom property wrapper
@propertyWrapper
struct Clamped<Value: Comparable> {
    var wrappedValue: Value {
        didSet { wrappedValue = min(max(wrappedValue, range.lowerBound), range.upperBound) }
    }
    let range: ClosedRange<Value>

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.wrappedValue = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
}

struct Volume {
    @Clamped(0...100) var level: Int = 50
}

var vol = Volume()
vol.level = 150  // clamped to 100
print(vol.level)  // 100

// @UserDefault wrapper
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }
}

class Settings {
    @UserDefault(key: "dark_mode", defaultValue: false)
    var isDarkMode: Bool
}
```

---

## Q8. Explain enums in Swift.

**Answer:**

```swift
// Enum with associated values
enum NetworkResult {
    case success(Data)
    case failure(Error)
    case loading(progress: Double)
}

switch result {
case .success(let data): process(data)
case .failure(let error): show(error)
case .loading(let progress) where progress > 0.5: showHalfway()
case .loading: showLoading()
}

// Enum with raw values
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
}

// Enum with methods and computed properties
enum Planet: Int, CaseIterable {
    case mercury = 1, venus, earth, mars

    var isHabitable: Bool { self == .earth }

    static var count: Int { allCases.count }
}

// Recursive enum
indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```

---

## Q9. What is the difference between `struct` and `class` in Swift?

**Answer:**

| Feature | Struct | Class |
|---------|--------|-------|
| Type | Value | Reference |
| Inheritance | ❌ | ✅ |
| Deinitializer | ❌ | ✅ |
| Memberwise init | Auto-generated | Manual |
| Identity (`===`) | ❌ | ✅ |
| Copy-on-Write | For stdlib collections | N/A |
| Thread safety | Safer (copies) | Needs sync |

**Apple's guidance:** Use structs unless you need class features. SwiftUI views are all structs.

---

## Q10. Explain `Result` type and error handling in Swift.

**Answer:**

```swift
// Throwing functions
enum NetworkError: Error {
    case invalidURL
    case noData
    case decodingFailed(Error)
}

func fetchUser(id: Int) throws -> User {
    guard let url = URL(string: "https://api.com/users/\(id)") else {
        throw NetworkError.invalidURL
    }
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Handling
do {
    let user = try fetchUser(id: 1)
} catch NetworkError.invalidURL {
    print("Bad URL")
} catch {
    print("Error: \(error)")
}

// Result type
func fetchUser(id: Int, completion: @escaping (Result<User, NetworkError>) -> Void) {
    // ...
    completion(.success(user))
    completion(.failure(.noData))
}

fetchUser(id: 1) { result in
    switch result {
    case .success(let user): print(user.name)
    case .failure(let error): print(error)
    }
}
```
