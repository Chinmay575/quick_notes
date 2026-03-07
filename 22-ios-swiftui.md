# 22 — iOS SwiftUI

---

## Q1. Explain SwiftUI fundamentals.

**Answer:**

```swift
struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 16) {
            Text("Count: \(count)")
                .font(.largeTitle)
                .foregroundColor(.blue)

            HStack {
                Button("Decrement") { count -= 1 }
                    .buttonStyle(.bordered)
                Button("Increment") { count += 1 }
                    .buttonStyle(.borderedProminent)
            }
        }
        .padding()
    }
}
```

**Key concepts:**
- Views are structs (value types) — lightweight, recreated often
- `body` is a computed property that returns the view hierarchy
- SwiftUI uses a diffing algorithm to update only changed parts
- Modifiers return new views (functional/chainable)

---

## Q2. Explain SwiftUI state management — @State, @Binding, @ObservedObject, @StateObject, @EnvironmentObject.

**Answer:**

```swift
// @State — local, owned by the view
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        Button("Count: \(count)") { count += 1 }
    }
}

// @Binding — two-way connection to parent's state
struct ToggleRow: View {
    @Binding var isOn: Bool  // doesn't own, just references
    var body: some View {
        Toggle("Setting", isOn: $isOn) // $ creates binding
    }
}

// @StateObject — owns the ObservableObject (created once)
struct UserListView: View {
    @StateObject private var viewModel = UserViewModel() // created once
    var body: some View {
        List(viewModel.users) { user in Text(user.name) }
    }
}

// @ObservedObject — does NOT own (passed from parent)
struct UserDetailView: View {
    @ObservedObject var viewModel: UserDetailViewModel // passed in
    var body: some View { Text(viewModel.user.name) }
}

// @EnvironmentObject — shared across view hierarchy
class AppSettings: ObservableObject {
    @Published var isDarkMode = false
    @Published var fontSize: CGFloat = 14
}

// Provide
ContentView().environmentObject(AppSettings())

// Access anywhere in tree
struct SettingsView: View {
    @EnvironmentObject var settings: AppSettings
    var body: some View {
        Toggle("Dark Mode", isOn: $settings.isDarkMode)
    }
}

// @Environment — system environment values
struct MyView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.dismiss) var dismiss
    @Environment(\.horizontalSizeClass) var sizeClass
}
```

| Wrapper | Owns? | For | Source |
|---------|-------|-----|--------|
| @State | Yes | Simple value types | Local |
| @Binding | No | Two-way to parent's State | Parent |
| @StateObject | Yes | ObservableObject | Local (init once) |
| @ObservedObject | No | ObservableObject | Parent |
| @EnvironmentObject | No | Shared ObservableObject | Ancestor |
| @Environment | No | System values | SwiftUI system |

---

## Q3. What is the `@Observable` macro (iOS 17+)?

**Answer:**

```swift
// Old way (ObservableObject + @Published)
class UserViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
}

// New way (@Observable macro, iOS 17+)
@Observable
class UserViewModel {
    var users: [User] = []
    var isLoading = false
    // No @Published needed!
    // Automatically tracks which properties each view reads
    // Only re-renders views that read changed properties
}

// Usage — no @ObservedObject/@StateObject needed for observation
struct UserListView: View {
    var viewModel: UserViewModel // just a regular property!
    var body: some View {
        if viewModel.isLoading {
            ProgressView()
        } else {
            List(viewModel.users) { user in Text(user.name) }
        }
    }
}

// For ownership, use @State
@State private var viewModel = UserViewModel()
```

---

## Q4. Explain SwiftUI Navigation (NavigationStack).

**Answer:**

```swift
// NavigationStack (iOS 16+)
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            List(users) { user in
                NavigationLink(value: user) {
                    Text(user.name)
                }
            }
            .navigationDestination(for: User.self) { user in
                UserDetailView(user: user)
            }
            .navigationDestination(for: Post.self) { post in
                PostDetailView(post: post)
            }
            .navigationTitle("Users")
        }
    }

    // Programmatic navigation
    func navigateToUser(_ user: User) {
        path.append(user)
    }

    func popToRoot() {
        path.removeLast(path.count)
    }
}

// TabView
TabView {
    HomeView()
        .tabItem { Label("Home", systemImage: "house") }
    SearchView()
        .tabItem { Label("Search", systemImage: "magnifyingglass") }
    ProfileView()
        .tabItem { Label("Profile", systemImage: "person") }
}
```

---

## Q5. How do you make network requests in SwiftUI?

**Answer:**

```swift
@Observable
class UserViewModel {
    var users: [User] = []
    var isLoading = false
    var error: String?

    func loadUsers() async {
        isLoading = true
        error = nil
        defer { isLoading = false }

        do {
            let (data, _) = try await URLSession.shared.data(
                from: URL(string: "https://api.example.com/users")!
            )
            users = try JSONDecoder().decode([User].self, from: data)
        } catch {
            self.error = error.localizedDescription
        }
    }
}

struct UserListView: View {
    @State private var viewModel = UserViewModel()

    var body: some View {
        List(viewModel.users) { user in
            Text(user.name)
        }
        .overlay {
            if viewModel.isLoading { ProgressView() }
        }
        .task {
            await viewModel.loadUsers() // called when view appears
        }
        .refreshable {
            await viewModel.loadUsers() // pull-to-refresh
        }
    }
}
```

---

## Q6. How do you create custom views and modifiers?

**Answer:**

```swift
// Custom View
struct GradientButton: View {
    let title: String
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            Text(title)
                .font(.headline)
                .foregroundColor(.white)
                .padding()
                .frame(maxWidth: .infinity)
                .background(
                    LinearGradient(colors: [.blue, .purple], startPoint: .leading, endPoint: .trailing)
                )
                .cornerRadius(12)
        }
    }
}

// Custom ViewModifier
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(Color(.systemBackground))
            .cornerRadius(12)
            .shadow(color: .gray.opacity(0.3), radius: 5, x: 0, y: 2)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}

// Usage
Text("Hello").cardStyle()
```

---

## Q7. Explain SwiftUI Lists and performance.

**Answer:**

```swift
// Basic List
List(users) { user in
    UserRow(user: user)
}

// Sections
List {
    Section("Active Users") {
        ForEach(activeUsers) { UserRow(user: $0) }
    }
    Section("Inactive Users") {
        ForEach(inactiveUsers) { UserRow(user: $0) }
    }
}

// Lazy loading (SwiftUI Lists are lazy by default)
// For grids:
ScrollView {
    LazyVGrid(columns: [GridItem(.adaptive(minimum: 150))]) {
        ForEach(items) { item in
            ItemCard(item: item)
        }
    }
}

// Searchable
.searchable(text: $searchText, prompt: "Search users")
.onChange(of: searchText) { _, newValue in
    viewModel.search(query: newValue)
}
```

---

## Q8. How do you handle forms and user input?

**Answer:**

```swift
struct RegistrationForm: View {
    @State private var name = ""
    @State private var email = ""
    @State private var password = ""
    @State private var agreedToTerms = false
    @State private var selectedRole = Role.user

    var body: some View {
        Form {
            Section("Personal Info") {
                TextField("Name", text: $name)
                TextField("Email", text: $email)
                    .keyboardType(.emailAddress)
                    .textContentType(.emailAddress)
                    .autocapitalization(.none)
                SecureField("Password", text: $password)
            }

            Section("Preferences") {
                Picker("Role", selection: $selectedRole) {
                    ForEach(Role.allCases) { role in
                        Text(role.rawValue).tag(role)
                    }
                }
                Toggle("I agree to terms", isOn: $agreedToTerms)
            }

            Section {
                Button("Register") { submit() }
                    .disabled(!isValid)
            }
        }
    }

    var isValid: Bool {
        !name.isEmpty && email.contains("@") && password.count >= 8 && agreedToTerms
    }
}
```

---

## Q9. How do you animate in SwiftUI?

**Answer:**

```swift
// Implicit animation
@State private var isExpanded = false

Circle()
    .frame(width: isExpanded ? 200 : 100)
    .animation(.spring(response: 0.5, dampingFraction: 0.6), value: isExpanded)
    .onTapGesture { isExpanded.toggle() }

// Explicit animation
withAnimation(.easeInOut(duration: 0.3)) {
    isExpanded.toggle()
}

// Transitions
if showDetail {
    DetailView()
        .transition(.asymmetric(
            insertion: .slide.combined(with: .opacity),
            removal: .opacity
        ))
}

// Matched geometry effect (Hero animation)
@Namespace private var animation

if !isExpanded {
    Image("photo")
        .matchedGeometryEffect(id: "photo", in: animation)
} else {
    Image("photo")
        .matchedGeometryEffect(id: "photo", in: animation)
        .frame(maxWidth: .infinity)
}
```
