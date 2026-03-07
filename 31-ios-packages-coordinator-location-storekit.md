# 31 — iOS Extras: Package Management, Coordinator, Core Location, AVFoundation, StoreKit, Background

---

## Q1. Compare CocoaPods, Swift Package Manager (SPM), and Carthage.

**Answer:**

| Feature | CocoaPods | SPM | Carthage |
|---------|-----------|-----|----------|
| **Maintained by** | Community | Apple (built into Xcode) | Community |
| **Config file** | Podfile (Ruby) | Package.swift | Cartfile |
| **Integration** | Modifies .xcworkspace | Built into Xcode | Pre-built frameworks |
| **Binary caching** | Partial | ✅ (Xcode 15+) | ✅ |
| **Build time** | Rebuilds all pods | Incremental | Pre-built = fast |
| **Monorepo support** | ✅ | ✅ | Limited |
| **Recommended** | Legacy projects | New projects (preferred) | Declining usage |

```ruby
# CocoaPods (Podfile)
platform :ios, '16.0'
use_frameworks!

target 'MyApp' do
  pod 'Alamofire', '~> 5.8'
  pod 'SnapKit'
end
# pod install → opens .xcworkspace
```

```swift
// SPM (Package.swift or Xcode → File → Add Package Dependencies)
let package = Package(
    name: "MyPackage",
    platforms: [.iOS(.v16)],
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    ],
    targets: [
        .target(name: "MyPackage", dependencies: ["Alamofire"]),
    ]
)
```

**Recommendation:** Use SPM for new projects (Apple's first-party, zero config, built into Xcode).

---

## Q2. Explain the Coordinator pattern in iOS.

**Answer:**

The Coordinator pattern separates navigation logic from view controllers, making them reusable and testable.

```swift
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    var navigationController: UINavigationController { get set }
    func start()
}

class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var navigationController: UINavigationController

    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }

    func start() {
        if isLoggedIn {
            showHome()
        } else {
            showAuth()
        }
    }

    private func showAuth() {
        let authCoordinator = AuthCoordinator(nav: navigationController)
        authCoordinator.onLoginSuccess = { [weak self] in
            self?.removeChild(authCoordinator)
            self?.showHome()
        }
        childCoordinators.append(authCoordinator)
        authCoordinator.start()
    }

    private func showHome() {
        let homeCoordinator = HomeCoordinator(nav: navigationController)
        childCoordinators.append(homeCoordinator)
        homeCoordinator.start()
    }

    private func removeChild(_ coordinator: Coordinator) {
        childCoordinators.removeAll { $0 === coordinator }
    }
}

class AuthCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var navigationController: UINavigationController
    var onLoginSuccess: (() -> Void)?

    init(nav: UINavigationController) { self.navigationController = nav }

    func start() {
        let loginVC = LoginViewController()
        loginVC.onLogin = { [weak self] in self?.onLoginSuccess?() }
        loginVC.onSignup = { [weak self] in self?.showSignup() }
        navigationController.pushViewController(loginVC, animated: true)
    }

    private func showSignup() {
        let signupVC = SignupViewController()
        navigationController.pushViewController(signupVC, animated: true)
    }
}
```

**Benefits:** ViewControllers don't know about each other → reusable, testable, clean.

---

## Q3. How do you use Core Location and MapKit?

**Answer:**

```swift
import CoreLocation
import MapKit

class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var onLocationUpdate: ((CLLocation) -> Void)?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.distanceFilter = 10 // meters
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
        // or manager.requestAlwaysAuthorization() for background
    }

    func startTracking() { manager.startUpdatingLocation() }
    func stopTracking() { manager.stopUpdatingLocation() }

    // Delegate methods
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        onLocationUpdate?(location)
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        switch manager.authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways: startTracking()
        case .denied, .restricted: showSettingsAlert()
        case .notDetermined: break
        @unknown default: break
        }
    }

    // Geocoding
    func reverseGeocode(_ location: CLLocation) async throws -> String {
        let geocoder = CLGeocoder()
        let placemarks = try await geocoder.reverseGeocodeLocation(location)
        return placemarks.first?.locality ?? "Unknown"
    }

    // Geofencing
    func monitorRegion(center: CLLocationCoordinate2D, radius: Double, id: String) {
        let region = CLCircularRegion(center: center, radius: radius, identifier: id)
        region.notifyOnEntry = true
        region.notifyOnExit = true
        manager.startMonitoring(for: region)
    }

    func locationManager(_ manager: CLLocationManager, didEnterRegion region: CLRegion) {
        print("Entered: \(region.identifier)")
    }
}

// Info.plist keys required:
// NSLocationWhenInUseUsageDescription
// NSLocationAlwaysUsageDescription (if needed)
```

---

## Q4. How do you work with AVFoundation for camera?

**Answer:**

```swift
import AVFoundation

class CameraManager: NSObject {
    private let session = AVCaptureSession()
    private var photoOutput = AVCapturePhotoOutput()
    private var currentDevice: AVCaptureDevice?

    func setup() async throws {
        // 1. Check permission
        switch AVCaptureDevice.authorizationStatus(for: .video) {
        case .authorized: break
        case .notDetermined:
            guard await AVCaptureDevice.requestAccess(for: .video) else { throw CameraError.denied }
        default: throw CameraError.denied
        }

        // 2. Configure session
        session.beginConfiguration()
        session.sessionPreset = .photo

        // 3. Add input (back camera)
        guard let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back) else {
            throw CameraError.noCamera
        }
        currentDevice = camera
        let input = try AVCaptureDeviceInput(device: camera)
        if session.canAddInput(input) { session.addInput(input) }

        // 4. Add output
        if session.canAddOutput(photoOutput) { session.addOutput(photoOutput) }

        session.commitConfiguration()
    }

    func startSession() {
        Task { session.startRunning() }
    }

    func capturePhoto(delegate: AVCapturePhotoCaptureDelegate) {
        let settings = AVCapturePhotoSettings()
        settings.flashMode = .auto
        photoOutput.capturePhoto(with: settings, delegate: delegate)
    }

    func switchCamera() throws {
        let newPosition: AVCaptureDevice.Position = (currentDevice?.position == .back) ? .front : .back
        // Remove current input, add new
    }

    // Preview layer for UI
    func previewLayer() -> AVCaptureVideoPreviewLayer {
        let layer = AVCaptureVideoPreviewLayer(session: session)
        layer.videoGravity = .resizeAspectFill
        return layer
    }
}
```

---

## Q5. How do you implement In-App Purchases with StoreKit 2?

**Answer:**

```swift
import StoreKit

class StoreManager: ObservableObject {
    @Published var products: [Product] = []
    @Published var purchasedProductIDs: Set<String> = []

    private let productIDs = ["premium_monthly", "premium_yearly", "remove_ads"]
    private var updateListener: Task<Void, Error>?

    init() {
        updateListener = listenForTransactions()
        Task { await loadProducts() }
    }

    func loadProducts() async {
        do {
            products = try await Product.products(for: productIDs)
                .sorted { $0.price < $1.price }
        } catch { print("Failed to load products: \(error)") }
    }

    func purchase(_ product: Product) async throws {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            let transaction = try checkVerified(verification)
            await updatePurchasedProducts()
            await transaction.finish()
        case .userCancelled: break
        case .pending: break // waiting for approval (Ask to Buy)
        @unknown default: break
        }
    }

    func restorePurchases() async {
        try? await AppStore.sync()
        await updatePurchasedProducts()
    }

    private func updatePurchasedProducts() async {
        var purchased = Set<String>()
        for await result in Transaction.currentEntitlements {
            if let transaction = try? checkVerified(result) {
                purchased.insert(transaction.productID)
            }
        }
        await MainActor.run { purchasedProductIDs = purchased }
    }

    private func listenForTransactions() -> Task<Void, Error> {
        Task.detached {
            for await result in Transaction.updates {
                if let transaction = try? self.checkVerified(result) {
                    await self.updatePurchasedProducts()
                    await transaction.finish()
                }
            }
        }
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .verified(let value): return value
        case .unverified: throw StoreError.verificationFailed
        }
    }
}

// SwiftUI usage
struct PaywallView: View {
    @StateObject var store = StoreManager()

    var body: some View {
        ForEach(store.products, id: \.id) { product in
            Button("Buy \(product.displayName) - \(product.displayPrice)") {
                Task { try await store.purchase(product) }
            }
        }
    }
}
```

---

## Q6. Explain iOS background modes and background processing.

**Answer:**

```swift
// Enable in Xcode: Signing & Capabilities → Background Modes
// Available modes:
// - Audio, AirPlay, and Picture in Picture
// - Location updates
// - External accessory communication
// - Background fetch
// - Remote notifications
// - Background processing

// 1. Background Tasks (BGTaskScheduler) — iOS 13+
import BackgroundTasks

// Register in AppDelegate or @main App
func application(_ application: UIApplication, didFinishLaunchingWithOptions...) {
    BGTaskScheduler.shared.register(
        forTaskWithIdentifier: "com.app.refresh",
        using: nil
    ) { task in
        self.handleAppRefresh(task: task as! BGAppRefreshTask)
    }

    BGTaskScheduler.shared.register(
        forTaskWithIdentifier: "com.app.db_cleanup",
        using: nil
    ) { task in
        self.handleDatabaseCleanup(task: task as! BGProcessingTask)
    }
}

func handleAppRefresh(task: BGAppRefreshTask) {
    scheduleAppRefresh() // schedule next one

    let operation = RefreshOperation()
    task.expirationHandler = { operation.cancel() }

    operation.completionBlock = {
        task.setTaskCompleted(success: !operation.isCancelled)
    }
    OperationQueue().addOperation(operation)
}

func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60) // 15 min
    try? BGTaskScheduler.shared.submit(request)
}

// 2. URLSession background download
let config = URLSessionConfiguration.background(withIdentifier: "com.app.download")
config.isDiscretionary = false
config.sessionSendsLaunchEvents = true
let session = URLSession(configuration: config, delegate: self, delegateQueue: nil)
let task = session.downloadTask(with: url)
task.resume()
```

**Background time limits:**
- App refresh: ~30 seconds
- Processing tasks: several minutes (device charging, Wi-Fi)
- Continuous background: location, audio, VoIP only

---

## Q7. Explain Size Classes and Trait Collections in iOS.

**Answer:**

```swift
// Size classes: Compact (.compact) or Regular (.regular)
// iPhone portrait:  width=compact, height=regular
// iPhone landscape: width=compact, height=compact (most)
// iPad:            width=regular,  height=regular

// UIKit
override func traitCollectionDidChange(_ previousTraitCollection: UITraitCollection?) {
    super.traitCollectionDidChange(previousTraitCollection)
    if traitCollection.horizontalSizeClass == .compact {
        // phone-like layout
    } else {
        // iPad / wide layout
    }
    if traitCollection.userInterfaceStyle == .dark {
        // dark mode
    }
}

// SwiftUI
struct AdaptiveView: View {
    @Environment(\.horizontalSizeClass) var horizontalSizeClass
    @Environment(\.verticalSizeClass) var verticalSizeClass
    @Environment(\.colorScheme) var colorScheme

    var body: some View {
        if horizontalSizeClass == .compact {
            VStack { content }   // stack vertically on phone
        } else {
            HStack { content }   // side by side on iPad
        }
    }
}

// ViewThatFits (iOS 16+) — automatic layout selection
ViewThatFits(in: .horizontal) {
    HStack { buttons }  // tries this first
    VStack { buttons }  // falls back to this
}
```

---

## Q8. Explain Objective-C interop with Swift.

**Answer:**

```swift
// Swift calling Objective-C:
// 1. Add ObjC files to project → Xcode creates bridging header
// 2. Import in bridging header: MyApp-Bridging-Header.h
//    #import "LegacyManager.h"
// 3. Use in Swift directly:
let manager = LegacyManager()
manager.fetchData()

// Objective-C calling Swift:
// 1. Import auto-generated header: #import "MyApp-Swift.h"
// 2. Swift classes must be marked:
@objc class UserManager: NSObject {
    @objc func getUser(id: String) -> User { ... }
    @objc dynamic var name: String = ""  // dynamic for KVO
}

// Key differences:
// - ObjC doesn't understand Swift structs, enums with associated values, generics (partially)
// - Use @objcMembers on class to expose all members
// - NS_SWIFT_NAME() for better Swift naming
// - Nullability annotations (nullable, nonnull) map to Swift optionals
```

---

## Q9. How do you handle Universal Links in iOS?

**Answer:**

```swift
// 1. Configure Apple App Site Association (AASA) file
// Host at: https://yourdomain.com/.well-known/apple-app-site-association
{
    "applinks": {
        "apps": [],
        "details": [{
            "appID": "TEAMID.com.example.myapp",
            "paths": ["/product/*", "/user/*", "/invite/*"]
        }]
    }
}

// 2. Add Associated Domains capability in Xcode
// applinks:yourdomain.com

// 3. Handle in UIKit (SceneDelegate)
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }
    handleDeepLink(url)
}

// 4. Handle in SwiftUI
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }
    }
}

func handleDeepLink(_ url: URL) {
    let pathComponents = url.pathComponents // ["/", "product", "123"]
    switch pathComponents.dropFirst().first {
    case "product": navigateToProduct(id: pathComponents.last ?? "")
    case "user": navigateToProfile(id: pathComponents.last ?? "")
    default: break
    }
}
```
