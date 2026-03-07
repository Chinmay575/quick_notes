# 24 — iOS Architecture & Testing

---

## Q1. Explain VIPER architecture.

**Answer:**

```
View ← Presenter → Interactor → Entity
                ↓
              Router
```

| Component | Responsibility |
|-----------|---------------|
| **View** | Display UI, forward user actions to Presenter |
| **Interactor** | Business logic, data fetching |
| **Presenter** | Mediator — formats data for View, handles View events |
| **Entity** | Data models |
| **Router** | Navigation logic |

```swift
// Protocols
protocol UserListViewProtocol: AnyObject {
    func showUsers(_ users: [UserViewModel])
    func showError(_ message: String)
}

protocol UserListPresenterProtocol {
    func viewDidLoad()
    func didSelectUser(_ user: UserViewModel)
}

protocol UserListInteractorProtocol {
    func fetchUsers() async throws -> [User]
}

protocol UserListRouterProtocol {
    func navigateToDetail(user: User)
}

// Presenter
class UserListPresenter: UserListPresenterProtocol {
    weak var view: UserListViewProtocol?
    var interactor: UserListInteractorProtocol
    var router: UserListRouterProtocol

    func viewDidLoad() {
        Task {
            do {
                let users = try await interactor.fetchUsers()
                let viewModels = users.map { UserViewModel(from: $0) }
                await MainActor.run { view?.showUsers(viewModels) }
            } catch {
                await MainActor.run { view?.showError(error.localizedDescription) }
            }
        }
    }
}
```

---

## Q2. Explain The Composable Architecture (TCA).

**Answer:**

```swift
import ComposableArchitecture

// Feature
@Reducer
struct CounterFeature {
    @ObservableState
    struct State: Equatable {
        var count = 0
        var isLoading = false
    }

    enum Action {
        case incrementButtonTapped
        case decrementButtonTapped
        case fetchCountResponse(Result<Int, Error>)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none
            case .decrementButtonTapped:
                state.count -= 1
                return .none
            case .fetchCountResponse(.success(let count)):
                state.count = count
                state.isLoading = false
                return .none
            case .fetchCountResponse(.failure):
                state.isLoading = false
                return .none
            }
        }
    }
}

// View
struct CounterView: View {
    let store: StoreOf<CounterFeature>

    var body: some View {
        VStack {
            Text("\(store.count)")
            Button("+") { store.send(.incrementButtonTapped) }
            Button("-") { store.send(.decrementButtonTapped) }
        }
    }
}
```

**Benefits:** Predictable state, testable, composable features, side effect management.

---

## Q3. How do you implement dependency injection in iOS?

**Answer:**

```swift
// 1. Constructor injection (preferred)
class UserViewModel {
    private let repository: UserRepositoryProtocol
    private let analytics: AnalyticsProtocol

    init(repository: UserRepositoryProtocol, analytics: AnalyticsProtocol) {
        self.repository = repository
        self.analytics = analytics
    }
}

// 2. Protocol-based with factory
protocol DependencyContainer {
    func makeUserRepository() -> UserRepositoryProtocol
    func makeAnalytics() -> AnalyticsProtocol
}

class AppContainer: DependencyContainer {
    func makeUserRepository() -> UserRepositoryProtocol { UserRepository(api: api) }
    func makeAnalytics() -> AnalyticsProtocol { FirebaseAnalytics() }
}

// 3. @Environment in SwiftUI
struct UserRepositoryKey: EnvironmentKey {
    static let defaultValue: UserRepositoryProtocol = UserRepository()
}
extension EnvironmentValues {
    var userRepository: UserRepositoryProtocol {
        get { self[UserRepositoryKey.self] }
        set { self[UserRepositoryKey.self] = newValue }
    }
}

// Provide
ContentView().environment(\.userRepository, MockUserRepository())

// 4. Swinject (DI framework)
let container = Container()
container.register(UserRepositoryProtocol.self) { _ in UserRepository() }
container.register(UserViewModel.self) { r in
    UserViewModel(repository: r.resolve(UserRepositoryProtocol.self)!)
}
```

---

## Q4. How do you write unit tests in Swift with XCTest?

**Answer:**

```swift
import XCTest
@testable import MyApp

class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!
    var mockRepository: MockUserRepository!

    override func setUp() {
        super.setUp()
        mockRepository = MockUserRepository()
        sut = UserViewModel(repository: mockRepository)
    }

    override func tearDown() {
        sut = nil
        mockRepository = nil
        super.tearDown()
    }

    func testLoadUsers_success() async {
        // Arrange
        mockRepository.stubbedUsers = [User(name: "Alice")]

        // Act
        await sut.loadUsers()

        // Assert
        XCTAssertEqual(sut.users.count, 1)
        XCTAssertEqual(sut.users.first?.name, "Alice")
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.error)
    }

    func testLoadUsers_failure() async {
        mockRepository.shouldFail = true
        await sut.loadUsers()
        XCTAssertTrue(sut.users.isEmpty)
        XCTAssertNotNil(sut.error)
    }

    func testPerformance() {
        measure {
            sut.processLargeDataSet()
        }
    }
}

// Mock
class MockUserRepository: UserRepositoryProtocol {
    var stubbedUsers: [User] = []
    var shouldFail = false

    func getUsers() async throws -> [User] {
        if shouldFail { throw NetworkError.noData }
        return stubbedUsers
    }
}
```

---

## Q5. How do you test SwiftUI views?

**Answer:**

```swift
// ViewInspector library
import ViewInspector

struct ContentView: View, Inspectable {
    @State var count = 0
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") { count += 1 }
        }
    }
}

func testContentView() throws {
    let view = ContentView()
    let text = try view.inspect().vStack().text(0).string()
    XCTAssertEqual(text, "Count: 0")
}

// Snapshot testing (swift-snapshot-testing)
import SnapshotTesting

func testUserCard() {
    let view = UserCard(user: .mock)
    assertSnapshot(of: view, as: .image(layout: .fixed(width: 375, height: 100)))
}

// UI tests with XCUITest
class LoginUITests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        continueAfterFailure = false
        app.launch()
    }

    func testLoginFlow() {
        let emailField = app.textFields["Email"]
        emailField.tap()
        emailField.typeText("test@test.com")

        let passwordField = app.secureTextFields["Password"]
        passwordField.tap()
        passwordField.typeText("password123")

        app.buttons["Login"].tap()

        XCTAssertTrue(app.staticTexts["Welcome"].waitForExistence(timeout: 5))
    }
}
```

---

## Q6. How do you test async code in Swift?

**Answer:**

```swift
// Async test methods
func testAsyncFetch() async throws {
    let users = try await sut.fetchUsers()
    XCTAssertFalse(users.isEmpty)
}

// Testing Combine publishers
import Combine

func testPublisher() {
    let expectation = expectation(description: "Publisher emits value")
    var cancellables = Set<AnyCancellable>()

    sut.usersPublisher
        .dropFirst() // skip initial value
        .sink { users in
            XCTAssertEqual(users.count, 2)
            expectation.fulfill()
        }
        .store(in: &cancellables)

    sut.loadUsers()
    wait(for: [expectation], timeout: 5.0)
}

// Testing with expectations
func testCallback() {
    let exp = expectation(description: "Callback called")
    sut.fetchData { result in
        switch result {
        case .success(let data): XCTAssertNotNil(data)
        case .failure: XCTFail("Should not fail")
        }
        exp.fulfill()
    }
    waitForExpectations(timeout: 5)
}
```
