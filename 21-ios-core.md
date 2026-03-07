# 21 — iOS Core (UIKit)

---

## Q1. Explain the iOS app lifecycle.

**Answer:**

```swift
// AppDelegate (pre-iOS 13 or non-scene apps)
@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_:didFinishLaunchingWithOptions:) -> Bool {
        // App launched — setup once
        return true
    }
    func applicationDidBecomeActive(_:) { /* foreground, interactive */ }
    func applicationWillResignActive(_:) { /* transitioning (call, notification) */ }
    func applicationDidEnterBackground(_:) { /* background — save state */ }
    func applicationWillEnterForeground(_:) { /* returning from background */ }
    func applicationWillTerminate(_:) { /* about to be killed */ }
}

// SceneDelegate (iOS 13+)
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    func scene(_:willConnectTo:options:) {
        // Set up window and root view controller
        window = UIWindow(windowScene: scene as! UIWindowScene)
        window?.rootViewController = UINavigationController(rootViewController: HomeVC())
        window?.makeKeyAndVisible()
    }
    func sceneDidBecomeActive(_:) { /* scene is active */ }
    func sceneWillResignActive(_:) { /* scene going inactive */ }
    func sceneDidEnterBackground(_:) { /* scene in background */ }
}
```

**App states:** Not Running → Inactive → Active → Background → Suspended

---

## Q2. Explain UIViewController lifecycle.

**Answer:**

```swift
class MyViewController: UIViewController {
    override func loadView() {
        // Create custom view (don't call super if providing custom view)
        view = MyCustomView()
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        // Called ONCE — setup UI, add subviews, configure
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        // About to appear — refresh data, start animations
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        // Fully visible — start timers, analytics
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        // About to leave — save state, stop timers
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        // No longer visible — cancel network requests
    }

    override func viewWillLayoutSubviews() { /* before layout */ }
    override func viewDidLayoutSubviews() { /* after layout — frames are set */ }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Release caches, images
    }
}
```

**Order:** `loadView` → `viewDidLoad` → `viewWillAppear` → `viewDidAppear` → ... → `viewWillDisappear` → `viewDidDisappear`

---

## Q3. Explain Auto Layout.

**Answer:**

```swift
// Programmatic constraints
let label = UILabel()
label.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(label)

NSLayoutConstraint.activate([
    label.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 16),
    label.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    label.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
])

// Content hugging & compression resistance
label.setContentHuggingPriority(.defaultHigh, for: .vertical)
label.setContentCompressionResistancePriority(.required, for: .vertical)

// Stack View (simplifies layout)
let stack = UIStackView(arrangedSubviews: [nameLabel, emailLabel, ageLabel])
stack.axis = .vertical
stack.spacing = 8
stack.alignment = .leading
stack.distribution = .fill
```

**Intrinsic Content Size:** Some views know their natural size (UILabel, UIButton). Use content hugging (resist growing) and compression resistance (resist shrinking) priorities.

---

## Q4. Explain UITableView and UICollectionView.

**Answer:**

```swift
// UITableView with Diffable Data Source (modern)
class UserListVC: UIViewController {
    enum Section { case main }

    var dataSource: UITableViewDiffableDataSource<Section, User>!
    let tableView = UITableView()

    override func viewDidLoad() {
        super.viewDidLoad()

        // Register cell
        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")

        // Diffable data source
        dataSource = UITableViewDiffableDataSource(tableView: tableView) { tableView, indexPath, user in
            let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
            var content = cell.defaultContentConfiguration()
            content.text = user.name
            content.secondaryText = user.email
            cell.contentConfiguration = content
            return cell
        }

        // Apply snapshot
        var snapshot = NSDiffableDataSourceSnapshot<Section, User>()
        snapshot.appendSections([.main])
        snapshot.appendItems(users, toSection: .main)
        dataSource.apply(snapshot, animatingDifferences: true)
    }
}

// UICollectionView with Compositional Layout
let layout = UICollectionViewCompositionalLayout { sectionIndex, environment in
    let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(0.5),
                                          heightDimension: .fractionalHeight(1.0))
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
    item.contentInsets = NSDirectionalEdgeInsets(top: 4, leading: 4, bottom: 4, trailing: 4)

    let groupSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1.0),
                                           heightDimension: .absolute(200))
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

    return NSCollectionLayoutSection(group: group)
}
```

---

## Q5. Explain delegation pattern in iOS.

**Answer:**

```swift
// 1. Define protocol
protocol UserSelectionDelegate: AnyObject {
    func didSelectUser(_ user: User)
    func didDeselectUser(_ user: User)
}

// 2. Use delegate in child
class UserPickerVC: UIViewController {
    weak var delegate: UserSelectionDelegate? // WEAK to avoid retain cycle

    func userTapped(_ user: User) {
        delegate?.didSelectUser(user)
        dismiss(animated: true)
    }
}

// 3. Conform in parent
class MainVC: UIViewController, UserSelectionDelegate {
    func showPicker() {
        let picker = UserPickerVC()
        picker.delegate = self
        present(picker, animated: true)
    }

    func didSelectUser(_ user: User) {
        print("Selected: \(user.name)")
    }

    func didDeselectUser(_ user: User) { }
}
```

---

## Q6. Explain GCD (Grand Central Dispatch).

**Answer:**

```swift
// Main queue (UI updates)
DispatchQueue.main.async {
    self.label.text = "Updated"
}

// Background queue
DispatchQueue.global(qos: .userInitiated).async {
    let data = self.processData()
    DispatchQueue.main.async {
        self.updateUI(with: data)
    }
}

// Quality of Service
// .userInteractive — highest, animation
// .userInitiated — user action, needs quick result
// .utility — long-running, progress bar
// .background — lowest, prefetch, backup

// Serial queue (tasks run one at a time)
let serialQueue = DispatchQueue(label: "com.app.serial")

// Concurrent queue
let concurrentQueue = DispatchQueue(label: "com.app.concurrent", attributes: .concurrent)

// DispatchGroup (wait for multiple tasks)
let group = DispatchGroup()
group.enter()
fetchUsers { users in self.users = users; group.leave() }
group.enter()
fetchPosts { posts in self.posts = posts; group.leave() }
group.notify(queue: .main) {
    self.updateUI() // called when ALL tasks complete
}

// Semaphore (limit concurrency)
let semaphore = DispatchSemaphore(value: 3) // max 3 concurrent
```

---

## Q7. What is the Responder Chain?

**Answer:**
The responder chain is a series of `UIResponder` objects that handle events (touches, gestures, keyboard).

```
UIView → UIViewController → UIWindow → UIApplication → AppDelegate
```

```swift
// Custom event handling
class MyView: UIView {
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        // Handle touch or pass to next responder
        super.touchesBegan(touches, with: event) // passes up the chain
    }

    override var canBecomeFirstResponder: Bool { true }
}

// First responder
textField.becomeFirstResponder()  // show keyboard
textField.resignFirstResponder()  // hide keyboard
```

---

## Q8. Explain NotificationCenter.

**Answer:**

```swift
// Post notification
NotificationCenter.default.post(
    name: .userDidLogin,
    object: nil,
    userInfo: ["userId": 42]
)

// Observe
let observer = NotificationCenter.default.addObserver(
    forName: .userDidLogin,
    object: nil,
    queue: .main
) { notification in
    let userId = notification.userInfo?["userId"] as? Int
    print("User \(userId ?? 0) logged in")
}

// Remove observer
NotificationCenter.default.removeObserver(observer)

// Custom notification name
extension Notification.Name {
    static let userDidLogin = Notification.Name("userDidLogin")
    static let cartUpdated = Notification.Name("cartUpdated")
}
```

---

## Q9. What are the different ways to pass data between view controllers?

**Answer:**

| Method | Direction | Use Case |
|--------|-----------|----------|
| Property injection | Forward | Push/present with data |
| Delegation | Backward | Return data to parent |
| Closure/callback | Backward | Simple one-off results |
| NotificationCenter | Any | Broadcast to multiple |
| UserDefaults | Any | Persistent settings |
| Singleton | Any | Shared state (use carefully) |

```swift
// Forward: property
let detailVC = DetailVC()
detailVC.user = selectedUser
navigationController?.pushViewController(detailVC, animated: true)

// Backward: closure
let pickerVC = ColorPickerVC()
pickerVC.onColorSelected = { [weak self] color in
    self?.view.backgroundColor = color
}
present(pickerVC, animated: true)
```
