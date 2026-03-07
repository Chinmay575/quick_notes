# 29 — Flutter Features: Firebase, Notifications, Permissions, Maps, Payments

---

## Q1. How do you implement Firebase Authentication in Flutter?

**Answer:**

```dart
// Setup: firebase_auth + google_sign_in
final auth = FirebaseAuth.instance;

// Email/Password signup
Future<User?> signUp(String email, String password) async {
  final credential = await auth.createUserWithEmailAndPassword(
    email: email, password: password,
  );
  await credential.user?.sendEmailVerification();
  return credential.user;
}

// Email/Password login
Future<User?> signIn(String email, String password) async {
  final credential = await auth.signInWithEmailAndPassword(
    email: email, password: password,
  );
  return credential.user;
}

// Google Sign-In
Future<User?> signInWithGoogle() async {
  final googleUser = await GoogleSignIn().signIn();
  final googleAuth = await googleUser?.authentication;
  final credential = GoogleAuthProvider.credential(
    accessToken: googleAuth?.accessToken,
    idToken: googleAuth?.idToken,
  );
  final result = await auth.signInWithCredential(credential);
  return result.user;
}

// Listen to auth state
auth.authStateChanges().listen((User? user) {
  if (user == null) { /* logged out */ }
  else { /* logged in */ }
});

// Sign out
await auth.signOut();
await GoogleSignIn().signOut();

// Password reset
await auth.sendPasswordResetEmail(email: email);
```

---

## Q2. How do you use Cloud Firestore in Flutter?

**Answer:**

```dart
final firestore = FirebaseFirestore.instance;

// Create
await firestore.collection('users').doc(uid).set({
  'name': 'Alice',
  'email': 'alice@test.com',
  'createdAt': FieldValue.serverTimestamp(),
});

// Read (once)
final doc = await firestore.collection('users').doc(uid).get();
final user = doc.data(); // Map<String, dynamic>

// Read (real-time stream)
firestore.collection('users').snapshots().listen((snapshot) {
  for (var doc in snapshot.docs) {
    print('${doc.id}: ${doc.data()}');
  }
});

// Query
final query = await firestore.collection('users')
    .where('age', isGreaterThan: 18)
    .orderBy('age')
    .limit(20)
    .get();

// Update
await firestore.collection('users').doc(uid).update({
  'name': 'Alice Updated',
  'updatedAt': FieldValue.serverTimestamp(),
});

// Delete
await firestore.collection('users').doc(uid).delete();

// Batch write
final batch = firestore.batch();
batch.set(docRef1, data1);
batch.update(docRef2, data2);
batch.delete(docRef3);
await batch.commit();

// Transaction
await firestore.runTransaction((transaction) async {
  final snapshot = await transaction.get(docRef);
  final newCount = snapshot.get('count') + 1;
  transaction.update(docRef, {'count': newCount});
});

// Pagination
Query query = firestore.collection('posts').orderBy('createdAt').limit(10);
// First page
final firstPage = await query.get();
// Next page
final nextPage = await query.startAfterDocument(firstPage.docs.last).get();
```

---

## Q3. How do you implement Push Notifications in Flutter (FCM)?

**Answer:**

```dart
// Setup: firebase_messaging + flutter_local_notifications

class NotificationService {
  final messaging = FirebaseMessaging.instance;
  final localNotifications = FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    // 1. Request permission
    final settings = await messaging.requestPermission(
      alert: true, badge: true, sound: true,
      provisional: false,
    );
    if (settings.authorizationStatus != AuthorizationStatus.authorized) return;

    // 2. Get FCM token
    final token = await messaging.getToken();
    print('FCM Token: $token');
    await saveTokenToServer(token!);

    // 3. Token refresh
    messaging.onTokenRefresh.listen(saveTokenToServer);

    // 4. Handle foreground messages
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      _showLocalNotification(message);
    });

    // 5. Handle background messages (must be top-level function)
    FirebaseMessaging.onBackgroundMessage(_backgroundHandler);

    // 6. Handle notification tap (app was in background)
    FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
      handleDeepLink(message.data);
    });

    // 7. Handle notification tap (app was terminated)
    final initialMessage = await messaging.getInitialMessage();
    if (initialMessage != null) handleDeepLink(initialMessage.data);

    // 8. Setup local notifications
    await localNotifications.initialize(
      InitializationSettings(
        android: AndroidInitializationSettings('@mipmap/ic_launcher'),
        iOS: DarwinInitializationSettings(),
      ),
      onDidReceiveNotificationResponse: (response) {
        handleDeepLink(jsonDecode(response.payload ?? '{}'));
      },
    );
  }

  void _showLocalNotification(RemoteMessage message) {
    localNotifications.show(
      message.hashCode,
      message.notification?.title,
      message.notification?.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          'default', 'Default',
          importance: Importance.high, priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      payload: jsonEncode(message.data),
    );
  }
}

// Top-level function (required for background handling)
@pragma('vm:entry-point')
Future<void> _backgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  print('Background message: ${message.messageId}');
}

// Topic subscription
await FirebaseMessaging.instance.subscribeToTopic('news');
await FirebaseMessaging.instance.unsubscribeFromTopic('news');
```

---

## Q4. How do you handle permissions in Flutter?

**Answer:**

```dart
// Using permission_handler package
import 'package:permission_handler/permission_handler.dart';

class PermissionService {
  // Check and request single permission
  Future<bool> requestCamera() async {
    var status = await Permission.camera.status;
    if (status.isGranted) return true;

    if (status.isDenied) {
      status = await Permission.camera.request();
      return status.isGranted;
    }

    if (status.isPermanentlyDenied) {
      // User denied permanently — open app settings
      await openAppSettings();
      return false;
    }
    return false;
  }

  // Request multiple permissions
  Future<Map<Permission, PermissionStatus>> requestMultiple() async {
    return await [
      Permission.camera,
      Permission.microphone,
      Permission.location,
    ].request();
  }

  // Check location permission with precision
  Future<bool> requestLocation() async {
    var status = await Permission.locationWhenInUse.status;
    if (!status.isGranted) {
      status = await Permission.locationWhenInUse.request();
    }
    // For background location (requires separate request on Android 10+)
    if (status.isGranted) {
      final bgStatus = await Permission.locationAlways.request();
      return bgStatus.isGranted;
    }
    return false;
  }
}

// Common permissions:
// Permission.camera, Permission.photos, Permission.microphone,
// Permission.location, Permission.locationAlways, Permission.locationWhenInUse,
// Permission.storage, Permission.contacts, Permission.calendar,
// Permission.notification, Permission.bluetooth
```

---

## Q5. How do you integrate Google Maps in Flutter?

**Answer:**

```dart
// google_maps_flutter + geolocator

class MapScreen extends StatefulWidget {
  @override
  State<MapScreen> createState() => _MapScreenState();
}

class _MapScreenState extends State<MapScreen> {
  GoogleMapController? _controller;
  final Set<Marker> _markers = {};
  LatLng _currentPosition = LatLng(28.6139, 77.2090); // Delhi

  @override
  void initState() {
    super.initState();
    _getCurrentLocation();
  }

  Future<void> _getCurrentLocation() async {
    final position = await Geolocator.getCurrentPosition(
      desiredAccuracy: LocationAccuracy.high,
    );
    setState(() {
      _currentPosition = LatLng(position.latitude, position.longitude);
      _markers.add(Marker(
        markerId: MarkerId('current'),
        position: _currentPosition,
        infoWindow: InfoWindow(title: 'You are here'),
      ));
    });
    _controller?.animateCamera(CameraUpdate.newLatLngZoom(_currentPosition, 15));
  }

  @override
  Widget build(BuildContext context) {
    return GoogleMap(
      initialCameraPosition: CameraPosition(target: _currentPosition, zoom: 12),
      onMapCreated: (controller) => _controller = controller,
      markers: _markers,
      myLocationEnabled: true,
      myLocationButtonEnabled: true,
      mapType: MapType.normal,
      onTap: (position) {
        setState(() {
          _markers.add(Marker(
            markerId: MarkerId(position.toString()),
            position: position,
          ));
        });
      },
    );
  }
}

// Real-time location tracking
StreamSubscription<Position>? _locationStream;
_locationStream = Geolocator.getPositionStream(
  locationSettings: LocationSettings(accuracy: LocationAccuracy.high, distanceFilter: 10),
).listen((Position position) {
  updateMarker(position);
});

// Geocoding (address ↔ coordinates)
// Using geocoding package
final placemarks = await placemarkFromCoordinates(lat, lng);
final locations = await locationFromAddress('1600 Amphitheatre Parkway');
```

---

## Q6. How do you implement In-App Purchases in Flutter?

**Answer:**

```dart
// Using in_app_purchase package
class PurchaseService {
  final InAppPurchase _iap = InAppPurchase.instance;
  late StreamSubscription<List<PurchaseDetails>> _subscription;

  Future<void> init() async {
    final available = await _iap.isAvailable();
    if (!available) return;

    // Listen to purchase updates
    _subscription = _iap.purchaseStream.listen(_onPurchaseUpdate);

    // Load products
    final response = await _iap.queryProductDetails({'premium_monthly', 'premium_yearly'});
    final products = response.productDetails;
  }

  void buyProduct(ProductDetails product) {
    final purchaseParam = PurchaseParam(productDetails: product);
    // Consumable (coins, gems)
    _iap.buyConsumable(purchaseParam: purchaseParam);
    // Non-consumable (premium, remove ads)
    _iap.buyNonConsumable(purchaseParam: purchaseParam);
  }

  void _onPurchaseUpdate(List<PurchaseDetails> purchases) {
    for (var purchase in purchases) {
      switch (purchase.status) {
        case PurchaseStatus.pending:
          // Show loading
          break;
        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          // Verify with server, then deliver content
          _verifyAndDeliver(purchase);
          break;
        case PurchaseStatus.error:
          // Show error
          break;
        case PurchaseStatus.canceled:
          break;
      }
      if (purchase.pendingCompletePurchase) {
        _iap.completePurchase(purchase);
      }
    }
  }

  // Restore purchases (required by Apple)
  Future<void> restorePurchases() async {
    await _iap.restorePurchases();
  }

  void dispose() { _subscription.cancel(); }
}
```

---

## Q7. Explain Theming in Flutter (Material 3, dark mode, custom themes).

**Answer:**

```dart
// Material 3 Theme
MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorSchemeSeed: Colors.blue,       // or colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue)
    brightness: Brightness.light,
    textTheme: GoogleFonts.poppinsTextTheme(),
    appBarTheme: AppBarTheme(
      centerTitle: true,
      elevation: 0,
    ),
    cardTheme: CardTheme(
      elevation: 2,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
    ),
    inputDecorationTheme: InputDecorationTheme(
      border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
      filled: true,
    ),
  ),
  darkTheme: ThemeData(
    useMaterial3: true,
    colorSchemeSeed: Colors.blue,
    brightness: Brightness.dark,
  ),
  themeMode: ThemeMode.system, // or .light / .dark
)

// Access theme values
final colorScheme = Theme.of(context).colorScheme;
final textTheme = Theme.of(context).textTheme;
Container(
  color: colorScheme.primaryContainer,
  child: Text('Hello', style: textTheme.headlineMedium),
)

// Custom theme extension
@immutable
class CustomColors extends ThemeExtension<CustomColors> {
  final Color? success;
  final Color? warning;
  const CustomColors({this.success, this.warning});

  @override
  CustomColors copyWith({Color? success, Color? warning}) =>
    CustomColors(success: success ?? this.success, warning: warning ?? this.warning);

  @override
  CustomColors lerp(ThemeExtension<CustomColors>? other, double t) {
    if (other is! CustomColors) return this;
    return CustomColors(
      success: Color.lerp(success, other.success, t),
      warning: Color.lerp(warning, other.warning, t),
    );
  }
}

// Add to theme
ThemeData(extensions: [CustomColors(success: Colors.green, warning: Colors.orange)])

// Access
Theme.of(context).extension<CustomColors>()!.success
```

---

## Q8. How do you implement Form Validation in Flutter?

**Answer:**

```dart
class LoginForm extends StatefulWidget {
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _obscurePassword = true;

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      autovalidateMode: AutovalidateMode.onUserInteraction,
      child: Column(
        children: [
          TextFormField(
            controller: _emailController,
            keyboardType: TextInputType.emailAddress,
            decoration: InputDecoration(labelText: 'Email', prefixIcon: Icon(Icons.email)),
            validator: (value) {
              if (value == null || value.isEmpty) return 'Email is required';
              if (!RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(value)) return 'Invalid email';
              return null; // valid
            },
          ),
          SizedBox(height: 16),
          TextFormField(
            controller: _passwordController,
            obscureText: _obscurePassword,
            decoration: InputDecoration(
              labelText: 'Password',
              prefixIcon: Icon(Icons.lock),
              suffixIcon: IconButton(
                icon: Icon(_obscurePassword ? Icons.visibility : Icons.visibility_off),
                onPressed: () => setState(() => _obscurePassword = !_obscurePassword),
              ),
            ),
            validator: (value) {
              if (value == null || value.isEmpty) return 'Password is required';
              if (value.length < 8) return 'Must be at least 8 characters';
              if (!value.contains(RegExp(r'[A-Z]'))) return 'Must contain uppercase';
              if (!value.contains(RegExp(r'[0-9]'))) return 'Must contain a number';
              return null;
            },
          ),
          SizedBox(height: 24),
          ElevatedButton(
            onPressed: () {
              if (_formKey.currentState!.validate()) {
                _formKey.currentState!.save();
                _submit();
              }
            },
            child: Text('Login'),
          ),
        ],
      ),
    );
  }

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
}
```

---

## Q9. Explain code generation in Flutter (build_runner, freezed).

**Answer:**

```dart
// freezed — immutable data classes + unions
@freezed
class User with _$User {
  const factory User({
    required int id,
    required String name,
    @Default('') String email,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

// Generates: copyWith, ==, hashCode, toString, fromJson, toJson
final user = User(id: 1, name: 'Alice');
final updated = user.copyWith(name: 'Bob');

// freezed union types (sealed)
@freezed
class AuthState with _$AuthState {
  const factory AuthState.initial() = _Initial;
  const factory AuthState.loading() = _Loading;
  const factory AuthState.authenticated(User user) = _Authenticated;
  const factory AuthState.error(String message) = _Error;
}

// Pattern matching
state.when(
  initial: () => LoginScreen(),
  loading: () => CircularProgressIndicator(),
  authenticated: (user) => HomeScreen(user: user),
  error: (msg) => ErrorScreen(message: msg),
);

// Run code generation
// dart run build_runner build --delete-conflicting-outputs
// dart run build_runner watch  (continuous)
```

---

## Q10. How do you manage packages and dependencies in Flutter?

**Answer:**

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.0         # Caret syntax (>=6.1.0 <7.0.0)
  http: '>=0.13.0 <2.0.0'  # Range
  my_package:
    git:
      url: https://github.com/user/repo.git
      ref: main
      path: packages/my_package  # monorepo
  local_package:
    path: ../local_package       # local path

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.0
  freezed: ^2.4.0
  mockito: ^5.4.0

dependency_overrides:
  http: ^1.1.0  # force version (use sparingly)
```

```bash
# Commands
flutter pub get              # install dependencies
flutter pub upgrade          # upgrade to latest compatible versions
flutter pub outdated         # show outdated packages
flutter pub deps             # show dependency tree
flutter pub add provider     # add dependency
flutter pub remove provider  # remove dependency
flutter pub publish          # publish your package
flutter pub cache clean      # clear cache
```

**Version resolution:** Dart uses `pub` solver — finds the newest versions satisfying all constraints. Conflicts show resolution errors.

---

## Q11. How do you implement Flutter Web and Desktop specifics?

**Answer:**

```dart
// Platform detection
import 'package:flutter/foundation.dart' show kIsWeb;
import 'dart:io' show Platform;

bool get isMobile => !kIsWeb && (Platform.isAndroid || Platform.isIOS);
bool get isDesktop => !kIsWeb && (Platform.isWindows || Platform.isMacOS || Platform.isLinux);
bool get isWeb => kIsWeb;

// Responsive breakpoints
class Responsive {
  static bool isMobileWidth(BuildContext context) => MediaQuery.of(context).size.width < 600;
  static bool isTabletWidth(BuildContext context) => MediaQuery.of(context).size.width < 1200;
  static bool isDesktopWidth(BuildContext context) => MediaQuery.of(context).size.width >= 1200;
}

// Conditional imports (web vs mobile)
// file: stub.dart → web_impl.dart / mobile_impl.dart
import 'stub.dart'
  if (dart.library.html) 'web_impl.dart'
  if (dart.library.io) 'mobile_impl.dart';

// Web-specific: hover effects, keyboard shortcuts, URL strategy
// Desktop-specific: window management, menu bar, system tray
```

**Key differences:**
- Web: No `dart:io`, use `dart:html`, URL-based navigation matters, SEO
- Desktop: Window management, keyboard shortcuts, menu bar, file system access
- Mouse vs touch: hover states, right-click, scroll wheel

---

## Q12. How do you implement background tasks and WorkManager in Flutter?

**Answer:**

```dart
// Using workmanager package
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    switch (task) {
      case 'syncData':
        await syncDataToServer();
        return true;
      case 'cleanCache':
        await cleanOldCache();
        return true;
    }
    return false;
  });
}

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  Workmanager().initialize(callbackDispatcher, isInDebugMode: false);

  // One-time task
  Workmanager().registerOneOffTask(
    'sync-1', 'syncData',
    constraints: Constraints(networkType: NetworkType.connected),
    inputData: {'key': 'value'},
  );

  // Periodic task (minimum 15 minutes on Android)
  Workmanager().registerPeriodicTask(
    'periodic-sync', 'syncData',
    frequency: Duration(hours: 1),
    constraints: Constraints(
      networkType: NetworkType.connected,
      requiresBatteryNotLow: true,
    ),
  );

  runApp(MyApp());
}
```
