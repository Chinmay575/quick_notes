# Flutter Senior Interview Revision Guide

This comprehensive guide covers all essential topics for a senior-level Flutter interview, including Flutter/Dart, Networking, DBMS, OS, and DSA basics. Each section includes explanations, deep dives, and examples.

---

**How to Use:**

- Skim "Key Points" for quick revision.
- Review "Pitfalls" to avoid common mistakes.
- Practice with "Sample Questions" and "Trick Questions" for each topic.
- Use code snippets and diagrams for hands-on understanding.

## 1. Flutter & Dart

### 1.1. Flutter Architecture

**Key Points:**

- Everything is a widget (including layout, padding, etc.).
- Widget tree → Element tree → Render tree.
- Stateless widgets are rebuilt when parent changes; Stateful widgets maintain state across rebuilds.
- Use keys for lists and dynamic widgets.
- Use const constructors for performance.
- Use InheritedWidget/Provider for efficient state propagation.

**Pitfalls:**

- Forgetting to use keys in lists can cause UI bugs.
- Overusing setState in large widgets causes unnecessary rebuilds.
- Misusing BuildContext (e.g., using context after async gap).

**Sample Questions:**

- Explain the difference between Stateless and Stateful widgets.
- How does Flutter render widgets to the screen?
- What is the role of BuildContext?

**Trick Questions:**

- What happens if you call setState after a widget is disposed?
- Can you access the parent widget’s state from a child? How?

**Explanation:**

Flutter uses a reactive UI framework. The UI is built using widgets, which describe what their view should look like given their current configuration and state.

**The Three Trees:**

1. **Widget Tree:** The immutable blueprint of the UI. Describes what should be displayed.
   - Widgets are lightweight and rebuilt frequently.
   - BuildContext ties widgets to their location in the tree.

2. **Element Tree:** The mutable, long-lived instance of widgets. Acts as a bridge between widgets and render objects.
   - Elements manage state and lifecycle.
   - Responsible for updating child elements.
   - One Element per Widget instance.

3. **Render Tree:** The visual representation of the UI. Handles layout and painting.
   - RenderObjects handle layout, painting, and hit testing.
   - Each Element has a corresponding RenderObject.
   - Responsible for rendering pixels to the screen.

**How It Works:**

- Widget describes configuration.
- Element manages the widget instance and state.
- RenderObject handles actual rendering.
- When state changes, Flutter rebuilds the Element and updates RenderObject only if necessary.

**Why Three Trees?**

- **Separation of concerns:** Widget tree is declarative; render tree is imperative.
- **Performance:** RenderObjects only updated when necessary.
- **Complexity management:** Easier to reason about UI changes.

**Lifecycle of a Widget:**

```
1. Widget created (immutable)
2. Element created and inserted into tree
3. RenderObject created and added to render tree
4. Layout phase (size and position calculated)
5. Paint phase (rendered to screen)
6. On state change, Element marks itself dirty
7. Framework rebuilds affected widgets
8. Dispose when widget removed from tree
```

**Stateless vs Stateful Widgets:**

| Aspect | Stateless | Stateful |
| --- | --- | --- |
| Mutability | Immutable | State can change |
| Rebuild Trigger | Parent rebuild | setState() or external change |
| Lifecycle | Simple (build) | Complex (initState, dispose, etc.) |
| Performance | Better (no state overhead) | Slightly slower |
| Use Cases | UI that doesn't change | Forms, animations, timers |

**Stateless Example:**

```dart
class MyText extends StatelessWidget {
 final String text;
 const MyText(this.text); // Should be const
 
 @override
 Widget build(BuildContext context) {
  return Text(text);
 }
}
```

**Stateful Example:

```dart
class Counter extends StatefulWidget {
 @override
 _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
 int count = 0;
 @override
 Widget build(BuildContext context) {
  return Column(
   children: [
    Text('$count'),
    ElevatedButton(
     onPressed: () => setState(() => count++),
     child: Text('Increment'),
    ),
   ],
  );
 }
}
```

**BuildContext:**

BuildContext is a reference to a widget's location in the widget tree. Key points:

- Used to access parent widgets and their properties.
- Provides access to Theme, MediaQuery, Navigator, etc.
- Should not be used after async gaps (e.g., after await).
- Never use context in an async callback; capture it before the async call.

```dart
// WRONG - context might be invalid after async operation
void fetchData() async {
 await Future.delayed(Duration(seconds: 1));
 Navigator.of(context).pop(); // May throw error
}

// CORRECT - use mounted check or capture before async
void fetchData() async {
 final nav = Navigator.of(context);
 await Future.delayed(Duration(seconds: 1));
 if (mounted) nav.pop();
}
```

**State Management Patterns:**

1. **InheritedWidget:** Low-level, rarely used directly. Base for other patterns.
2. **Provider:** Simple, powerful, recommended for most apps.
3. **Bloc:** For complex business logic separation.
4. **Riverpod:** Modern, functional approach.
5. **Redux:** For predictable state management.
6. **MobX:** Reactive state management.

**Provider Example (Recommended):**

```dart
class CounterModel extends ChangeNotifier {
 int count = 0;
 
 void increment() {
  count++;
  notifyListeners(); // Notify UI to rebuild
 }
}

// In widget tree
ChangeNotifierProvider(
 create: (context) => CounterModel(),
 child: Consumer<CounterModel>(
  builder: (context, counter, _) {
   return Text('Count: ${counter.count}');
  },
 ),
)
```

**Navigation:**

- **Navigator 1.0 (Imperative):** Direct method calls like push, pop.
- **Navigator 2.0 (Declarative):** More suitable for web and complex routing.

```dart
// Navigator 1.0
Navigator.of(context).push(MaterialPageRoute(builder: (_) => NextPage()));
Navigator.of(context).pop();

// Navigator 2.0 (with go_router package)
context.go('/home/details/123');
```

**Platform Channels:**

For calling native code (Java/Kotlin/Swift/ObjC):

```dart
// Flutter side
const platform = MethodChannel('com.example.app/battery');
final int result = await platform.invokeMethod('getBatteryLevel');

// Native side (Kotlin)
val channel = MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "com.example.app/battery")
channel.setMethodCallHandler { call, result ->
 if (call.method == "getBatteryLevel") {
  val level = getBatteryPercentage()
  result.success(level)
 }
}
```

**Hot Reload vs Hot Restart:**

- **Hot Reload:** Injects updated code into Dart VM. State is preserved. ~200ms.
- **Hot Restart:** Restarts the app from scratch. State is lost. ~1-2s.

Use Hot Reload for quick UI iterations; use Hot Restart after package changes or significant logic changes.

- Widget tree, Element tree, Render tree
- Stateless vs Stateful widgets
- BuildContext, InheritedWidget, Provider, Bloc, Riverpod
- Navigation (Navigator 1.0 & 2.0)
- State management patterns
- Custom widgets, themes, and animations
- Platform channels (MethodChannel, EventChannel)
- Hot reload vs hot restart

- Null safety, Futures, Streams, async/await
- Mixins, extensions, generics
- Isolates, concurrency
- Packages & pubspec.yaml

### 1.2. Dart Language

**Key Points:**

- Null safety is enforced at compile time.
- Futures and Streams are core to async programming.
- Mixins allow code reuse without inheritance.
- Extensions add methods to existing types.
- Isolates provide true parallelism (not just async).

**Pitfalls:**

- Forgetting to handle nulls leads to runtime errors.
- Blocking the main isolate with heavy computation causes UI jank.

**Sample Questions:**

- What is the difference between Future and Stream?
- How does Dart’s null safety work?
- How do you create a custom extension method?

**Trick Questions:**

- Can you await inside a synchronous function?
- What happens if you mutate a list while iterating over it?

**Dart Language Core Concepts:**

Dart is a modern, object-oriented language optimized for building fast apps on any platform.

**Null Safety (Sound Nullability):**

Dart enforces null safety at compile time to prevent null reference errors.

```dart
String? name; // Nullable - can be null or String
String greeting = 'Hello'; // Non-nullable - must be String

// Safe navigation
String? user = getUserName();
int length = user?.length ?? 0; // Use ?? for default value

// Null coalescing assignment
name ??= 'Guest'; // Assign only if name is null

// Force unwrap (dangerous)
String value = user!; // Throws if user is null
```

**Futures and Async/Await:**

Futures represent the result of an async operation that may complete at some point in the future.

```dart
// Without async/await
Future<String> fetchData() {
 return Future.delayed(Duration(seconds: 1))
  .then((_) => 'Data loaded');
}

// With async/await (preferred)
Future<String> fetchData() async {
 await Future.delayed(Duration(seconds: 1));
 return 'Data loaded';
}

// Error handling
try {
 String result = await fetchData();
 print(result);
} catch (e) {
 print('Error: $e');
} finally {
 print('Done');
}
```

**Streams:**

Streams emit a sequence of values over time. Used for real-time data.

```dart
// Creating a stream
Stream<int> countStream() async* {
 for (int i = 0; i < 5; i++) {
  yield i;
  await Future.delayed(Duration(seconds: 1));
 }
}

// Using streams
countStream().listen((value) => print(value));

// Broadcasting stream (multiple listeners)
final broadcast = countStream().asBroadcastStream();
broadcast.listen((value) => print('Listener 1: $value'));
broadcast.listen((value) => print('Listener 2: $value'));
```

**Mixins:**

Code reuse without inheritance. A class can use multiple mixins.

```dart
mixin Logger {
 void log(String msg) => print('[Log] $msg');
}

mixin Validator {
 bool isValid(String input) => input.isNotEmpty;
}

class User with Logger, Validator {
 String name;
 
 User(this.name) {
  if (isValid(name)) {
   log('User created: $name');
  }
 }
}
```

**Extensions:**

Add methods to existing types without inheritance.

```dart
extension StringCasing on String {
 String capitalize() => this[0].toUpperCase() + substring(1);
 String toTitleCase() => split(' ').map((e) => e.capitalize()).join(' ');
}

void main() {
 print('hello world'.capitalize()); // Hello world
 print('hello world'.toTitleCase()); // Hello World
}
```

**Generics:**

Type-safe collections and classes.

```dart
// Generic class
class Box<T> {
 T content;
 Box(this.content);
 T getContent() => content;
}

// Usage
Box<String> stringBox = Box('Hello');
Box<int> intBox = Box(42);

// Generic constraints
class Repository<T extends BaseModel> {
 void save(T item) => print('Saving ${item.id}');
}
```

**Isolates:**

For true parallelism. Each isolate is an isolated Dart execution context.

```dart
import 'dart:isolate';

void heavyComputation(SendPort sendPort) {
 int result = fibonacci(40);
 sendPort.send(result);
}

void main() async {
 final receivePort = ReceivePort();
 await Isolate.spawn(heavyComputation, receivePort.sendPort);
 final result = await receivePort.first;
 print('Result: $result');
}
```

**Packages & pubspec.yaml:**

- `pubspec.yaml` manages dependencies.
- Use `pub.dev` to find packages.
- Version constraints: `^1.2.3` (up to 2.0.0), `~1.2.3` (up to 1.3.0).

```yaml
dependencies:
 flutter:
  sdk: flutter
 provider: ^6.0.0
 http: ^1.0.0

dev_dependencies:
 flutter_test:
  sdk: flutter
 mockito: ^5.0.0
```

- Null safety, Futures, Streams, async/await
- Mixins, extensions, generics
- Isolates, concurrency
- Packages & pubspec.yaml

- Widget rebuilds, keys, const constructors
- Lazy loading, image caching
- Profiling tools (DevTools, Observatory)
- Reducing jank, optimizing build methods

### 1.3. Performance & Optimization

**Key Points:**

- Use const widgets and constructors where possible.
- Use ListView.builder for large lists.
- Profile with DevTools to find bottlenecks.
- Use image caching and lazy loading for performance.
- Minimize widget rebuilds by splitting widgets.

**Pitfalls:**

- Not using const leads to unnecessary rebuilds.
- Doing heavy work in build methods.
- Not disposing controllers (e.g., AnimationController, TextEditingController).

**Sample Questions:**

- How do you profile a Flutter app?
- What are keys and why are they important?
- How do you optimize list performance?

**Trick Questions:**

- What happens if you use a GlobalKey in multiple places?
- Can you use setState in a StatelessWidget?

**Explanation:**
Optimizing Flutter apps involves minimizing unnecessary widget rebuilds, using keys, and leveraging const constructors.

**Detailed Overview:**

Performance optimization in Flutter involves reducing frame times, minimizing memory usage, and ensuring smooth 60/120 fps rendering.

**1. Widget Rebuilds (The Heart of Performance):**

Every time `setState()` is called or parent rebuilds, all child widgets rebuild by default. This can cascade and cause jank.

```dart
// BAD - whole tree rebuilds
class BadParent extends StatefulWidget {
 @override
 State<BadParent> createState() => _BadParentState();
}

class _BadParentState extends State<BadParent> {
 int count = 0;
 
 @override
 Widget build(BuildContext context) {
  return Column(
   children: [
    Text('Count: $count'),
    const ExpensiveWidget(), // Should be const!
   ],
  );
 }
}
```

**2. Keys for Dynamic Lists:**

Widgets need keys when list items can change position.

```dart
// With keys - Flutter knows which widget corresponds to which data
ListView(
 children: items.map((item) => MyItem(key: ValueKey(item.id), item: item)).toList(),
)
```

**3. Const Constructors:**

Marked as `const` to reduce widget creation overhead.

```dart
class MyWidget extends StatelessWidget {
 final String text;
 const MyWidget(this.text);
 
 @override
 Widget build(BuildContext context) => Text(text);
}
```

**4. Lazy Loading with ListView.builder:**

```dart
// GOOD - loads items on demand
ListView.builder(
 itemCount: 1000,
 itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
)
```

**5. Image Caching & Optimization:**

```dart
// Using cached_network_image package
CachedNetworkImage(
 imageUrl: 'https://example.com/image.jpg',
 cacheWidth: 200,
 cacheHeight: 200,
 placeholder: (context, url) => CircularProgressIndicator(),
)
```

**6. Profiling with DevTools:**

- Frame rendering time (target < 16.67ms for 60fps)
- Memory usage
- Jank detection
- Run: `flutter pub global activate devtools && devtools`

**7. Reducing Jank - Use Isolates:**

```dart
// WRONG - blocks UI
List<int> result = List.generate(10000000, (i) => i * i);

// CORRECT - use Isolate
final result = await compute(expensiveFunction, param);
```

**8. Optimizing Build Methods:**

Cache expensive operations outside build().

```dart
late final expensiveData = expensiveQuery();

@override
Widget build(BuildContext context) {
 return ListView.builder(
  itemCount: expensiveData.length,
  itemBuilder: (context, index) => Text('${expensiveData[index]}'),
 );
}
```

**Advanced Patterns:**

- Use `RepaintBoundary` to optimize painting.
- Use `SingleChildScrollView` vs `ListView.builder` wisely.
- Use `KeepAlive` to maintain widget state in scrollable lists.

- **Widget rebuilds:** Use `const` widgets and avoid rebuilding large widget trees.
- **Keys:** Help Flutter identify widgets uniquely in lists.
- **Lazy loading:** Use `ListView.builder` for large lists.
- **Image caching:** Use `CachedNetworkImage` package.
- **Profiling tools:** Use Flutter DevTools for performance analysis.
- **Reducing jank:** Avoid heavy computation in the UI thread; use Isolates.
- Widget rebuilds, keys, const constructors
- Lazy loading, image caching
- Profiling tools (DevTools, Observatory)
- Reducing jank, optimizing build methods

- Unit, widget, and integration testing
- Mocking, test coverage

### 1.4. Testing

**Key Points:**

- **Unit Testing:** Test logic independent of UI.
- **Widget Testing:** Test UI components in isolation with mocked dependencies.
- **Integration Testing:** Test full app flows on a device/emulator.
- Coverage target: At least 80% for critical paths.
- Use mockito for mocking, fake for stubs.

**Detailed Overview:**

**1. Unit Testing:**

Tests pure functions and business logic.

```dart
// The function to test
int add(int a, int b) => a + b;

// Unit test
void main() {
 test('add function returns correct sum', () {
  expect(add(2, 3), equals(5));
  expect(add(-1, 1), equals(0));
  expect(add(0, 0), equals(0));
 });
}
```

**2. Widget Testing:**

Tests UI widgets without running on a device.

```dart
void main() {
 testWidgets('Counter increments', (WidgetTester tester) async {
  await tester.pumpWidget(MyApp());
  
  expect(find.text('0'), findsOneWidget);
  
  await tester.tap(find.byIcon(Icons.add));
  await tester.pump(); // Trigger rebuild
  
  expect(find.text('1'), findsOneWidget);
 });
}
```

**3. Integration Testing:**

Tests full app flows.

```dart
void main() {
 testWidgets('User can login', (WidgetTester tester) async {
  await tester.pumpWidget(MyApp());
  
  // Find and fill login form
  await tester.enterText(find.byType(TextField).first, 'user@example.com');
  await tester.enterText(find.byType(TextField).last, 'password');
  
  // Tap login button
  await tester.tap(find.byType(ElevatedButton));
  await tester.pumpAndSettle(); // Wait for animations
  
  expect(find.text('Home'), findsOneWidget);
 });
}
```

**4. Mocking with mockito:**

```dart
class MockHttpClient extends Mock implements HttpClient {}

void main() {
 test('fetchData returns data on success', () async {
  final mockClient = MockHttpClient();
  
  when(mockClient.get(any)).thenAnswer((_) async => Response(
   '{"name": "John"}',
   200,
  ));
  
  final result = await fetchUserData(mockClient);
  expect(result.name, 'John');
 });
}
```

**5. Test Coverage:**

```bash
# Generate coverage
flutter test --coverage

# View coverage (requires lcov)
lcov --list coverage/lcov.info
```

**Best Practices:**

- Test behavior, not implementation details.
- Use descriptive test names.
- Keep tests focused and independent.
- Use setUp() and tearDown() for common setup/cleanup.
- Aim for 80%+ coverage on critical code.

- Unit, widget, and integration testing
- Mocking, test coverage

- CI/CD, code signing, flavors
- App size reduction, obfuscation
- Play Store/App Store requirements

### 1.5. Deployment

**Key Points:**

- Automate builds and tests with CI/CD pipelines.
- Use flavors for managing multiple environments (dev, staging, prod).
- Code obfuscation protects intellectual property.
- Code signing is mandatory for app stores.
- Test release builds locally before uploading.

**Detailed Overview:**

**1. Flavors (Build Configurations):**

Support different environments without code changes.

```yaml
# pubspec.yaml - no changes needed for flavors

# Run with flavor
# flutter run --flavor dev
# flutter build apk --flavor prod
```

```dart
// main.dart - handle flavors
void main() {
 const flavor = String.fromEnvironment('FLAVOR', defaultValue: 'dev');
 runApp(MyApp(flavor: flavor));
}
```

**2. CI/CD Setup:**

Example GitHub Actions:

```yaml
name: Build and Test

on:
 push:
  branches: [main]
 pull_request:
  branches: [main]

jobs:
 build:
  runs-on: ubuntu-latest
  
  steps:
   - uses: actions/checkout@v2
   - uses: subosito/flutter-action@v2
   - run: flutter pub get
   - run: flutter analyze
   - run: flutter test
   - run: flutter build apk --release
```

**3. Code Signing:**

```bash
# Android - Create keystore
keytool -genkey -v -keystore ~/key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias key

# Sign APK
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore ~/key.jks app-release-unsigned.apk key

# Align APK
zipalign -v 4 app-release-unsigned.apk app-release.apk
```

**4. App Size Reduction:**

```bash
# Obfuscation and tree shaking
flutter build apk --obfuscate --split-debug-info=build/debug

# Analyze APK
flutter build apk
# Check with: bundletool analyze-bundle --bundle=build/app/outputs/bundle/release/app-release.aab
```

**5. Versioning:**

```yaml
# pubspec.yaml
version: 1.0.0+1  # version+build number
```

Update before each release:

- Patch version (1.0.1): Bug fixes
- Minor version (1.1.0): New features (backward compatible)
- Major version (2.0.0): Breaking changes

**6. Play Store Release:**

1. Create release keystore
2. Sign APK/AAB
3. Create app on Google Play Console
4. Upload AAB (Android App Bundle)
5. Configure store listing, privacy policy
6. Gradual rollout (5% → 25% → 100%)

**7. App Store Release (iOS):**

1. Create App ID in Apple Developer
2. Configure signing in Xcode
3. Create build on TestFlight
4. Gather beta testers and feedback
5. Submit for review
6. Wait for App Review (24-48 hours typically)
7. Release when approved

**Best Practices:**

- Use environment files for secrets (never commit API keys).
- Always test release builds before uploading.
- Versioning should follow semantic versioning.
- Use staged rollouts to catch issues early.
- Monitor crash reports and user feedback post-release.
- CI/CD, code signing, flavors
- App size reduction, obfuscation
- Play Store/App Store requirements

---

- HTTP, REST, GraphQL basics
- Packages: `http`, `dio`, `chopper`, `graphql_flutter`
- JSON serialization (manual, `json_serializable`)
- Error handling, retries, timeouts
- WebSockets, real-time communication
- Secure storage, OAuth, JWT, API keys
- Caching strategies

## 2. Networking

**Key Points:**

- Use `dio` for advanced networking with interceptors and retries.
- Always handle errors and timeouts explicitly.
- Never hardcode API URLs or store credentials in code.
- Use secure storage for tokens (flutter_secure_storage).
- Implement caching strategies for offline support.
- Validate and sanitize all API responses.

**Detailed Overview:**

**1. HTTP Clients:**

**http package (Simple):**

```dart
import 'package:http/http.dart' as http;

Future<void> fetchData() async {
 try {
  final response = await http.get(Uri.parse('https://api.example.com/data'));
  
  if (response.statusCode == 200) {
   print('Data: ${response.body}');
  } else {
   throw Exception('Failed to load data: ${response.statusCode}');
  }
 } catch (e) {
  print('Error: $e');
 }
}
```

**dio package (Advanced):**

```dart
import 'package:dio/dio.dart';

final dio = Dio()
 ..options.baseUrl = 'https://api.example.com'
 ..options.connectTimeout = Duration(seconds: 5)
 ..options.receiveTimeout = Duration(seconds: 5)
 ..interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
   print('Requesting ${options.path}');
   return handler.next(options);
  },
  onResponse: (response, handler) {
   print('Response: ${response.statusCode}');
   return handler.next(response);
  },
  onError: (error, handler) {
   print('Error: ${error.message}');
   return handler.next(error);
  },
 ));

Future<void> fetchWithDio() async {
 try {
  Response response = await dio.get('/data');
  print(response.data);
 } catch (e) {
  print('Error: $e');
 }
}
```

**2. JSON Serialization:**

**Manual (Not Recommended):**

```dart
class User {
 final String name;
 final int age;
 
 User({required this.name, required this.age});
 
 factory User.fromJson(Map<String, dynamic> json) {
  return User(
   name: json['name'] as String,
   age: json['age'] as int,
  );
 }
 
 Map<String, dynamic> toJson() => {
  'name': name,
  'age': age,
 };
}
```

**json_serializable (Recommended):**

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart';

@JsonSerializable()
class User {
 final String name;
 final int age;
 
 User({required this.name, required this.age});
 
 factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
 Map<String, dynamic> toJson() => _$UserToJson(this);
}

// Run: flutter pub run build_runner build
```

**3. Error Handling & Retries:**

```dart
Future<T> retryRequest<T>(
 Future<T> Function() request, {
 int maxRetries = 3,
 Duration delay = const Duration(seconds: 1),
}) async {
 for (int i = 0; i < maxRetries; i++) {
  try {
   return await request();
  } catch (e) {
   if (i == maxRetries - 1) rethrow;
   await Future.delayed(delay * (i + 1)); // Exponential backoff
  }
 }
 throw Exception('Max retries exceeded');
}

// Usage
final data = await retryRequest(() => fetchData());
```

**4. WebSockets for Real-time Data:**

```dart
import 'package:web_socket_channel/web_socket_channel.dart';

class WebSocketService {
 late WebSocketChannel channel;
 
 void connect() {
  channel = WebSocketChannel.connect(
   Uri.parse('wss://echo.websocket.org'),
  );
 }
 
 Stream get messages => channel.stream;
 
 void sendMessage(String message) {
  channel.sink.add(message);
 }
 
 void close() {
  channel.sink.close();
 }
}

// Usage
final ws = WebSocketService();
ws.connect();
ws.messages.listen((message) {
 print('Received: $message');
});
ws.sendMessage('Hello');
```

**5. Secure Token Storage:**

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenService {
 static const _storage = FlutterSecureStorage();
 
 Future<void> saveToken(String token) async {
  await _storage.write(key: 'auth_token', value: token);
 }
 
 Future<String?> getToken() async {
  return await _storage.read(key: 'auth_token');
 }
 
 Future<void> deleteToken() async {
  await _storage.delete(key: 'auth_token');
 }
}
```

**6. OAuth 2.0 & JWT:**

```dart
import 'package:oauth2/oauth2.dart' as oauth2;

Future<oauth2.Client> getOAuthClient() async {
 final grant = oauth2.AuthorizationCodeGrant(
  clientId,
  authorizationUrl,
  tokenUrl,
  secret: clientSecret,
 );
 
 // Redirect user to login, handle callback
 return await grant.handleAuthorizationResponse({...});
}

// JWT - Use google_sign_in or firebase_auth for authentication
final GoogleSignInAccount? account = await GoogleSignIn().signIn();
final GoogleSignInAuthentication auth = await account!.authentication;
final token = auth.idToken; // JWT token
```

**7. API Caching Strategies:**

```dart
class CachedApiClient {
 final Dio dio;
 final Duration cacheDuration;
 final Map<String, CacheEntry> _cache = {};
 
 CachedApiClient(this.dio, {this.cacheDuration = const Duration(minutes: 5)});
 
 Future<T> get<T>(String url, T Function(dynamic) parser) async {
  if (_cache.containsKey(url)) {
   final entry = _cache[url]!;
   if (DateTime.now().difference(entry.timestamp) < cacheDuration) {
    return entry.data as T;
   }
  }
  
  final response = await dio.get(url);
  final data = parser(response.data);
  _cache[url] = CacheEntry(data, DateTime.now());
  return data;
 }
}

class CacheEntry {
 final dynamic data;
 final DateTime timestamp;
 
 CacheEntry(this.data, this.timestamp);
}
```

**Best Practices:**

- Use environment variables for API URLs (dev, staging, prod).
- Implement token refresh logic for OAuth.
- Handle network timeouts gracefully.
- Validate SSL certificates in production.
- Use interceptors for adding auth headers.
- HTTP, REST, GraphQL basics
- Packages: `http`, `dio`, `chopper`, `graphql_flutter`
- JSON serialization (manual, `json_serializable`)
- Error handling, retries, timeouts
- WebSockets, real-time communication
- Secure storage, OAuth, JWT, API keys
- Caching strategies

---

- Relational (SQLite, PostgreSQL, MySQL)
- NoSQL (Firebase, Hive, MongoDB)
- ACID properties, transactions
- Indexing, normalization, denormalization
- ORM (Object-Relational Mapping)
- Query optimization
- Data migration, backup, and restore
- CAP theorem

## 3. Database Management Systems (DBMS)

**Key Points:**

- Choose database based on data model: relational (SQL) vs document-based (NoSQL).
- ACID properties ensure data integrity in relational DBs.
- Use indexes to speed up queries significantly.
- Normalization reduces redundancy; denormalization improves performance.
- Transactions ensure atomic operations (all-or-nothing).

**Detailed Overview:**

**1. Relational Databases:**

**SQLite (Local Storage):**

```dart
import 'package:sqflite/sqflite.dart';

class DatabaseHelper {
 static final DatabaseHelper instance = DatabaseHelper._init();
 static Database? _database;
 
 DatabaseHelper._init();
 
 Future<Database> get database async {
  if (_database != null) return _database!;
  _database = await _initDB('users.db');
  return _database!;
 }
 
 Future<Database> _initDB(String filePath) async {
  final dbPath = await getDatabasesPath();
  final path = join(dbPath, filePath);
  
  return await openDatabase(
   path,
   version: 1,
   onCreate: _createDB,
  );
 }
 
 Future<void> _createDB(Database db, int version) async {
  await db.execute('''
   CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   )
  ''');
  
  // Create index for faster queries
  await db.execute('''
   CREATE INDEX idx_email ON users(email)
  ''');
 }
 
 // CRUD operations
 Future<int> insertUser(User user) async {
  final db = await database;
  return await db.insert('users', user.toMap());
 }
 
 Future<User?> getUserById(int id) async {
  final db = await database;
  final result = await db.query(
   'users',
   where: 'id = ?',
   whereArgs: [id],
  );
  if (result.isNotEmpty) {
   return User.fromMap(result.first);
  }
  return null;
 }
 
 Future<List<User>> getAllUsers() async {
  final db = await database;
  final result = await db.query('users');
  return result.map((map) => User.fromMap(map)).toList();
 }
 
 Future<int> updateUser(User user) async {
  final db = await database;
  return await db.update(
   'users',
   user.toMap(),
   where: 'id = ?',
   whereArgs: [user.id],
  );
 }
 
 Future<int> deleteUser(int id) async {
  final db = await database;
  return await db.delete(
   'users',
   where: 'id = ?',
   whereArgs: [id],
  );
 }
 
 // Transactions
 Future<void> transaction() async {
  final db = await database;
  await db.transaction((txn) async {
   await txn.insert('users', {'name': 'John', 'email': 'john@example.com'});
   await txn.insert('users', {'name': 'Jane', 'email': 'jane@example.com'});
  });
 }
}
```

**2. NoSQL Databases:**

**Firebase Firestore (Cloud):**

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class FirestoreService {
 final FirebaseFirestore _firestore = FirebaseFirestore.instance;
 
 // Create document
 Future<void> createUser(String id, Map<String, dynamic> userData) async {
  await _firestore.collection('users').doc(id).set(userData);
 }
 
 // Read document
 Future<DocumentSnapshot> getUser(String id) async {
  return await _firestore.collection('users').doc(id).get();
 }
 
 // Real-time listener
 Stream<QuerySnapshot> getUsersStream() {
  return _firestore.collection('users').snapshots();
 }
 
 // Query
 Future<QuerySnapshot> getUsersByName(String name) async {
  return await _firestore
   .collection('users')
   .where('name', isEqualTo: name)
   .get();
 }
 
 // Update
 Future<void> updateUser(String id, Map<String, dynamic> updates) async {
  await _firestore.collection('users').doc(id).update(updates);
 }
 
 // Delete
 Future<void> deleteUser(String id) async {
  await _firestore.collection('users').doc(id).delete();
 }
 
 // Batch write
 Future<void> batchWrite() async {
  final batch = _firestore.batch();
  
  final ref1 = _firestore.collection('users').doc('user1');
  batch.set(ref1, {'name': 'Alice'});
  
  final ref2 = _firestore.collection('users').doc('user2');
  batch.update(ref2, {'name': 'Bob'});
  
  await batch.commit();
 }
}
```

**Hive (Local NoSQL):**

```dart
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';

@HiveType(typeId: 0)
class User {
 @HiveField(0)
 late String name;
 
 @HiveField(1)
 late String email;
}

Future<void> hiveOperations() async {
 await Hive.initFlutter();
 Hive.registerAdapter(UserAdapter());
 
 final box = await Hive.openBox<User>('users');
 
 // Create
 await box.add(User()..name = 'John'..email = 'john@example.com');
 
 // Read
 final user = box.getAt(0);
 
 // Update
 await box.putAt(0, User()..name = 'Jane'..email = 'jane@example.com');
 
 // Delete
 await box.deleteAt(0);
}
```

**3. ACID Properties:**

- **Atomicity:** All-or-nothing: transaction succeeds completely or fails completely.
- **Consistency:** DB moves from one valid state to another.
- **Isolation:** Concurrent transactions don't interfere.
- **Durability:** Committed data survives failures.

**4. Query Optimization:**

```dart
// SLOW - Full table scan
SELECT * FROM users WHERE name = 'John';

// FAST - Uses index
CREATE INDEX idx_name ON users(name);
SELECT * FROM users WHERE name = 'John';

// SLOW - No index
SELECT * FROM users WHERE age > 25 AND city = 'NYC';

// FAST - Composite index
CREATE INDEX idx_age_city ON users(age, city);
```

**5. Normalization:**

Organize data to eliminate redundancy:

```
UNNORMALIZED:
| id | name | city | product | price |
| 1  | John | NYC  | Laptop  | 1000  |
| 1  | John | NYC  | Phone   | 500   |

NORMALIZED (3NF):
Users:        |id|name |city|
              |1 |John |NYC |
              
Orders:       |id|user_id|product|price|
              |1 |1      |Laptop |1000 |
              |2 |1      |Phone  |500  |
```

**6. Data Migration:**

```dart
// Handle schema upgrades
Future<void> onUpgrade(Database db, int oldVersion, int newVersion) async {
 if (oldVersion < 2) {
  await db.execute('ALTER TABLE users ADD COLUMN phone TEXT');
 }
}
```

**Best Practices:**

- Choose database based on your access patterns.
- Use indexes wisely (improves reads, slows writes).
- Normalize for data integrity, denormalize for performance.
- Use transactions for related operations.
- Always handle database initialization errors.
- Relational (SQLite, PostgreSQL, MySQL)
- NoSQL (Firebase, Hive, MongoDB)
- ACID properties, transactions
- Indexing, normalization, denormalization
- ORM (Object-Relational Mapping)
- Query optimization
- Data migration, backup, and restore
- CAP theorem

---

- Process vs thread, multitasking
- Memory management (heap, stack, garbage collection)
- Deadlocks, race conditions, synchronization
- File systems, permissions
- Scheduling algorithms
- Inter-process communication (IPC)
- Virtualization, containers

## 4. Operating Systems (OS)

**Key Points:**

- **Process:** Independent program with its own memory space.
- **Thread:** Lightweight process; shares memory with other threads in same process.
- **Memory:** Stack (function calls, fast), Heap (dynamic allocation, slower).
- **Deadlock:** Occurs when processes wait for resources held by each other.
- **Race Condition:** Multiple threads access shared data unsynchronized.

**Detailed Overview:**

**1. Process vs Thread:**

| Aspect | Process | Thread |
| --- | --- | --- |
| Memory | Own isolated memory | Shared memory within process |
| Creation | Slow, expensive | Fast, lightweight |
| Communication | IPC (pipes, sockets) | Shared memory (needs synchronization) |
| Crash Impact | Isolated | Can crash entire process |
| Context Switch | Heavy | Lighter |

**2. Memory Management:**

**Stack:**

- Automatic, fast memory allocation.
- LIFO (Last In, First Out) structure.
- Memory freed automatically when function returns.
- Size is limited (stack overflow possible).

```
Function calls:
main() calls foo() which calls bar()

STACK:
[bar() local vars]
[foo() local vars]
[main() local vars]
```

**Heap:**

- Manual allocation (or via garbage collection).
- Available entire program lifetime.
- Slower than stack.
- Risk of memory leaks if not freed.

**Garbage Collection:**

- Automatic memory management in modern languages (Dart, Java, Python).
- Periodically frees unreferenced objects.
- Can cause pause (GC pause) affecting performance.

**3. Deadlock:**

**Example:**

```
Thread 1:
 1. Lock resourceA
 2. Wait for resourceB (held by Thread 2)

Thread 2:
 1. Lock resourceB
 2. Wait for resourceA (held by Thread 1)

Result: Both threads wait forever (DEADLOCK)
```

**Prevention:**

- **Lock Ordering:** Always acquire locks in the same order.
- **Timeout:** Use lock timeouts.
- **Banker's Algorithm:** For resource allocation safety.

**4. Race Conditions:**

**Example (Dart):**

```dart
class Counter {
 int count = 0;
 
 void increment() {
  count++; // NOT atomic - 3 operations: read, increment, write
 }
}

// If two threads call increment() simultaneously:
// Expected: 2
// Actual: 1 (race condition)
```

**Prevention:**

```dart
// Solution: Synchronization
class SafeCounter {
 int count = 0;
 final Mutex _mutex = Mutex(); // Simulated mutex
 
 Future<void> increment() async {
  // Lock prevents other threads from entering
  count++;
 }
}
```

**5. Scheduling Algorithms:**

- **Round Robin:** Each process gets equal time slice.
- **Priority Scheduling:** Higher priority processes run first.
- **FIFO (First Come, First Served):** Simple but not optimal.
- **Shortest Job First (SJF):** Minimize average wait time.
- **Multi-level Queue:** Different priority levels with queues.

**6. Inter-Process Communication (IPC):**

**Pipes:**

- One-way communication between processes.
- Used for piping commands: `command1 | command2`.

**Sockets:**

- Network communication between processes (local or remote).
- TCP/UDP sockets for data transmission.

**Shared Memory:**

- Direct memory sharing between processes.
- Fast but requires synchronization.
- Risk of race conditions.

**7. File Systems & Permissions:**

**Linux Permissions (Unix):**

```
drwxr-xr-x  user  group  file
| | | | |
| | | | others: execute, read, write
| | | group: read, write, execute  
| | owner: read, write, execute
| directory
total bits

chmod 755 file   # rwxr-xr-x
chmod 644 file   # rw-r--r--
```

**8. Virtualization & Containers:**

**Virtual Machines:**

- Full OS virtualization, heavy resource usage.
- Complete isolation.

**Containers (Docker):**

- OS-level virtualization, lightweight.
- Share kernel with host.
- Portable across environments.

```dockerfile
FROM debian:bullseye
RUN apt-get install -y flutter
COPY app /app
WORKDIR /app
CMD ["flutter", "run"]
```

**9. Key OS Concepts in Mobile:**

- **Battery Management:** Aggressive CPU/network throttling.
- **Memory Pressure:** Killing background apps when low.
- **Background Execution:** Limited background work allowed.
- **Permissions:** Android/iOS require explicit user permissions.

**Best Practices:**

- Use thread pools instead of creating threads manually.
- Always use proper synchronization for shared resources.
- Avoid blocking the main thread (UI freezes).
- Understand your app's memory footprint.
- Minimize threads and processes for battery efficiency.
- Process vs thread, multitasking
- Memory management (heap, stack, garbage collection)
- Deadlocks, race conditions, synchronization
- File systems, permissions
- Scheduling algorithms
- Inter-process communication (IPC)
- Virtualization, containers

---

- Arrays, linked lists, stacks, queues
- Trees (binary, BST, AVL, heap, trie)
- Graphs (BFS, DFS, shortest path)
- Hashing, hash tables
- Sorting (quick, merge, bubble, insertion)
- Searching (binary, linear)
- Big O notation, time/space complexity
- Recursion, dynamic programming

## 5. Data Structures & Algorithms (DSA) Basics

**Key Points:**

- Choose data structures based on access patterns.
- Master sorting algorithms: O(n log n) is standard (Quicksort, Mergesort).
- Understand recursion and base cases.
- Dynamic programming optimizes overlapping subproblems.
- Know space/time complexity tradeoffs.

**Detailed Overview:**

**1. Arrays & Lists:**

```dart
// Fixed size
List<int> arr = [1, 2, 3, 4, 5];
arr[0]; // O(1) access
arr.add(6); // O(1) amortized

// Time complexities:
// Access: O(1)
// Insert: O(n)
// Delete: O(n)
// Search: O(n) or O(log n) if sorted
```

**2. Linked Lists:**

```dart
class Node<T> {
 T data;
 Node<T>? next;
 Node(this.data);
}

class LinkedList<T> {
 Node<T>? head;
 
 // Insert at beginning: O(1)
 void insertFirst(T value) {
  final newNode = Node(value);
  newNode.next = head;
  head = newNode;
 }
 
 // Search: O(n)
 bool contains(T value) {
  Node<T>? current = head;
  while (current != null) {
   if (current.data == value) return true;
   current = current.next;
  }
  return false;
 }
 
 // Reverse: O(n)
 void reverse() {
  Node<T>? prev, curr = head;
  while (curr != null) {
   final next = curr.next;
   curr.next = prev;
   prev = curr;
   curr = next;
  }
  head = prev;
 }
}

// Time complexities:
// Access: O(n)
// Insert: O(1) at head, O(n) at position
// Delete: O(n)
// Search: O(n)
```

**3. Stacks (LIFO):**

```dart
class Stack<T> {
 final List<T> _stack = [];
 
 void push(T value) => _stack.add(value);
 T pop() => _stack.removeLast();
 T peek() => _stack.last;
 bool isEmpty() => _stack.isEmpty;
 int size() => _stack.length;
}

// Use cases:
// - Function call stack
// - Expression evaluation
// - Backtracking (undo/redo)
// - Browser back button
```

**4. Queues (FIFO):**

```dart
class Queue<T> {
 final List<T> _queue = [];
 
 void enqueue(T value) => _queue.add(value);
 T dequeue() => _queue.removeAt(0);
 T peek() => _queue.first;
 bool isEmpty() => _queue.isEmpty;
}

// Use cases:
// - Breadth-first search
// - Task scheduling
// - Print spooling
```

**5. Trees:**

**Binary Tree:**

```dart
class TreeNode<T> {
 T val;
 TreeNode<T>? left, right;
 TreeNode(this.val);
}

// Traversals:
// In-order (Left, Root, Right) - sorted for BST
void inOrder(TreeNode? node) {
 if (node == null) return;
 inOrder(node.left);
 print(node.val);
 inOrder(node.right);
}

// Pre-order (Root, Left, Right) - copy tree
void preOrder(TreeNode? node) {
 if (node == null) return;
 print(node.val);
 preOrder(node.left);
 preOrder(node.right);
}

// Post-order (Left, Right, Root) - delete tree
void postOrder(TreeNode? node) {
 if (node == null) return;
 postOrder(node.left);
 postOrder(node.right);
 print(node.val);
}

// Level-order (BFS) - breadth-first
void levelOrder(TreeNode? root) {
 if (root == null) return;
 final queue = Queue();
 queue.add(root);
 
 while (queue.isNotEmpty) {
  final node = queue.removeFirst();
  print(node.val);
  if (node.left != null) queue.add(node.left);
  if (node.right != null) queue.add(node.right);
 }
}
```

**Binary Search Tree (BST):**

```dart
class BST {
 TreeNode<int>? root;
 
 void insert(int val) {
  root = _insertHelper(root, val);
 }
 
 TreeNode<int>? _insertHelper(TreeNode<int>? node, int val) {
  if (node == null) return TreeNode(val);
  
  if (val < node.val) {
   node.left = _insertHelper(node.left, val);
  } else if (val > node.val) {
   node.right = _insertHelper(node.right, val);
  }
  return node;
 }
 
 // Search: O(log n) avg, O(n) worst
 bool search(int val) => _searchHelper(root, val);
 
 bool _searchHelper(TreeNode<int>? node, int val) {
  if (node == null) return false;
  if (node.val == val) return true;
  if (val < node.val) return _searchHelper(node.left, val);
  return _searchHelper(node.right, val);
 }
}
```

**6. Graphs:**

```dart
// Adjacency list representation
class Graph {
 final Map<int, List<int>> adjList = {};
 
 void addEdge(int u, int v) {
  adjList.putIfAbsent(u, () => []).add(v);
  adjList.putIfAbsent(v, () => []).add(u);
 }
 
 // BFS: O(V + E)
 void bfs(int start) {
  final visited = <int>{};
  final queue = Queue<int>();
  
  visited.add(start);
  queue.add(start);
  
  while (queue.isNotEmpty) {
   final node = queue.removeFirst();
   print(node);
   
   for (final neighbor in adjList[node] ?? []) {
    if (!visited.contains(neighbor)) {
     visited.add(neighbor);
     queue.add(neighbor);
    }
   }
  }
 }
 
 // DFS: O(V + E)
 void dfs(int node, {required Set<int> visited}) {
  visited.add(node);
  print(node);
  
  for (final neighbor in adjList[node] ?? []) {
   if (!visited.contains(neighbor)) {
    dfs(neighbor, visited: visited);
   }
  }
 }
 
 // Shortest path (Dijkstra): O((V + E) log V)
 // Cycle detection, topological sort, etc.
}
```

**7. Hash Tables:**

```dart
// O(1) average lookup
Map<String, int> map = {};

map['apple'] = 5;     // O(1)
map['banana'] = 3;    // O(1)
final count = map['apple']; // O(1)
map.remove('banana'); // O(1)

// Collision handling:
// - Chaining: Use linked list for collisions
// - Open addressing: Find next empty slot
```

**8. Sorting Algorithms:**

```dart
// Quicksort: O(n log n) avg, O(n^2) worst
void quicksort(List<int> arr, int low, int high) {
 if (low < high) {
  final pivot = partition(arr, low, high);
  quicksort(arr, low, pivot - 1);
  quicksort(arr, pivot + 1, high);
 }
}

// Mergesort: O(n log n) always, stable
void mergesort(List<int> arr, int left, int right) {
 if (left < right) {
  final mid = (left + right) ~/ 2;
  mergesort(arr, left, mid);
  mergesort(arr, mid + 1, right);
  merge(arr, left, mid, right);
 }
}

// Comparison:
// Quicksort: Faster in practice, but unstable
// Mergesort: Stable, but uses more space
// Heapsort: O(n log n) worst, in-place
```

**9. Searching:**

```dart
// Binary search: O(log n) - array must be sorted
int binarySearch(List<int> arr, int target) {
 int left = 0, right = arr.length - 1;
 
 while (left <= right) {
  final mid = (left + right) ~/ 2;
  
  if (arr[mid] == target) return mid;
  if (arr[mid] < target) {
   left = mid + 1;
  } else {
   right = mid - 1;
  }
 }
 return -1; // Not found
}

// Linear search: O(n)
```

**10. Recursion & Base Cases:**

```dart
// Factorial: O(n) time, O(n) space (call stack)
int factorial(int n) {
 if (n <= 1) return 1; // Base case
 return n * factorial(n - 1); // Recursive case
}

// Fibonacci (exponential without memoization)
int fib(int n) {
 if (n <= 1) return n;
 return fib(n - 1) + fib(n - 2);
}

// Fibonacci with memoization: O(n) time, O(n) space
Map<int, int> memo = {};

int fibMemo(int n) {
 if (memo.containsKey(n)) return memo[n]!;
 if (n <= 1) return n;
 
 memo[n] = fibMemo(n - 1) + fibMemo(n - 2);
 return memo[n]!;
}
```

**11. Dynamic Programming:**

```dart
// Coin change problem: O(n * m) time, O(m) space
int coinChange(List<int> coins, int amount) {
 final dp = List.filled(amount + 1, amount + 1);
 dp[0] = 0;
 
 for (int i = 1; i <= amount; i++) {
  for (final coin in coins) {
   if (coin <= i) {
    dp[i] = min(dp[i], dp[i - coin] + 1);
   }
  }
 }
 
 return dp[amount] == amount + 1 ? -1 : dp[amount];
}
```

**12. Big O Complexity:**

| Algorithm | Best | Average | Worst | Space |
| --- | --- | --- | --- | --- |
| Quicksort | O(n log n) | O(n log n) | O(n^2) | O(log n) |
| Mergesort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | O(1) |
| Bubble Sort | O(n) | O(n^2) | O(n^2) | O(1) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |

**Best Practices:**

- Understand time and space complexity.
- Choose right data structure for your use case.
- Practice recursion until it feels natural.
- Use memoization to optimize recursive problems.
- Always consider edge cases and base cases.
- Arrays, linked lists, stacks, queues
- Trees (binary, BST, AVL, heap, trie)
- Graphs (BFS, DFS, shortest path)
- Hashing, hash tables
- Sorting (quick, merge, bubble, insertion)
- Searching (binary, linear)
- Big O notation, time/space complexity
- Recursion, dynamic programming

---

- Scalability, reliability, maintainability
- Caching, load balancing
- API design, versioning
- Microservices vs monoliths
- Security best practices
- Logging, monitoring, analytics

## 6. System Design (Mobile Focus)

**Key Points:**

- Design for scalability, reliability, and maintainability.
- Plan for growth: 10x users should not break architecture.
- Distribute load with caching and load balancing.
- Monitor and log for observability.
- Security should be designed, not bolted on.

**Detailed Overview:**

**1. Core Principles:**

**Scalability:**

- Horizontal: Add more servers/instances.
- Vertical: Add more resources to existing server.
- Database sharding: Distribute data across multiple databases.

**Reliability (Availability):**

- Target 99.9% uptime (4.5 hours downtime/year).
- Redundancy: Duplicate critical components.
- Failover: Automatic switch to backup.

**Maintainability:**

- Modular architecture (microservices).
- Clear APIs and contracts.
- Documentation and monitoring.

**2. Caching Strategies:**

**Cache Hierarchy:**

```
Browser Cache → CDN → Server Cache → Database
```

**Types:**

- **Client-side:** Browser, localStorage.
- **Server-side:** Redis, Memcached.
- **CDN:** Distributed globally for static content.

**Cache Invalidation Strategies:**

- TTL (Time To Live): Expire after duration.
- LRU (Least Recently Used): Remove oldest unused.
- Event-based: Invalidate on data change.

**3. Load Balancing:**

```
Incoming Requests → Load Balancer
                    ├→ Server 1
                    ├→ Server 2
                    └→ Server 3
```

**Algorithms:**

- Round Robin: Equal distribution.
- Least Connections: Send to server with fewest active connections.
- Weighted: Prioritize capable servers.
- IP Hash: Same user always goes to same server (for session consistency).

**4. API Design (RESTful):**

```dart
// GOOD API Design
GET    /users              // List all users
POST   /users              // Create user
GET    /users/{id}         // Get specific user
PUT    /users/{id}         // Update user
DELETE /users/{id}         // Delete user

// Versioning
GET /api/v1/users          // Version in URL
GET /users (with Accept-Version header)

// Pagination
GET /users?page=1&limit=20

// Filtering
GET /users?status=active&role=admin

// Sorting
GET /users?sort=-created_at (descending)
```

**5. Database Optimization:**

**Denormalization:**

- Store redundant data for faster reads.
- Accept write complexity for read speed.

```
Users table:        Orders table:
[id][name][city]    [id][user_id][user_name][user_city][amount]
                     User name & city duplicated for faster queries
```

**Read Replicas:**

- Multiple read-only copies.
- Master handles writes.
- Slaves handle reads.

**6. Microservices vs Monolith:**

**Monolith:**

- Single codebase, single deployment.
- Easier to develop initially.
- Harder to scale individual components.

**Microservices:**

- Independent services with their own DBs.
- Scalable per service.
- Operational complexity (service discovery, inter-service communication).

**7. Event-Driven Architecture:**

```
User Service → Event Bus → Email Service
               ├→ Notification Service
               ├→ Analytics Service
```

Uses: Order placed → send email, update inventory, record analytics.

**8. Security:**

**Authentication:**

- OAuth 2.0, OpenID Connect, JWT, SAML.

**Authorization:**

- Role-based (RBAC): User has role, role has permissions.
- Attribute-based (ABAC): Fine-grained rules.

**Data Protection:**

- Encryption in transit (HTTPS/TLS).
- Encryption at rest (database encryption).
- Salting and hashing passwords.

**9. Monitoring & Observability:**

**Metrics:**

- Response time, throughput, error rates.
- CPU, memory, disk usage.
- Database query performance.

**Logging:**

- Structured logging (JSON format).
- Centralized logging (ELK stack, Splunk).

**Tracing:**

- Distributed tracing for request flow.
- Services: Jaeger, Zipkin.

**Tools:**

- Prometheus (metrics), Grafana (visualization).
- Sentry, Firebase Crashlytics (error tracking).
- DataDog, New Relic (APM).

**10. Mobile-Specific Design:**

**Offline-First:**

- Local cache with sync when online.
- Optimistic updates.

**Battery Efficiency:**

- Batch network requests.
- Adaptive sync intervals.
- Background processing limits.

**Network Efficiency:**

- Compression (gzip).
- Delta sync (only changed data).
- Pagination for large datasets.

**Best Practices:**

- Design for failures (graceful degradation).
- Use feature flags for gradual rollouts.
- Implement circuit breakers for external services.
- Plan for 10x growth without redesign.
- Scalability, reliability, maintainability
- Caching, load balancing
- API design, versioning
- Microservices vs monoliths
- Security best practices
- Logging, monitoring, analytics

---

- Project architecture decisions
- Mentoring, code reviews
- Agile, Scrum, Kanban
- Conflict resolution, stakeholder communication

## 7. Behavioral & Leadership

**Key Points:**

- Communicate clearly; be ready to explain trade-offs.
- Listen actively to team feedback and suggestions.
- Make and document technical decisions.
- Mentor junior developers and share knowledge.
- Handle conflicts constructively.

**Detailed Overview:**

**1. Technical Decision-Making:**

**Framework:**

1. **Define Problem:** Clearly state constraints and goals.
2. **Explore Options:** Consider 2-3 viable solutions.
3. **Compare Trade-offs:** Speed vs maintainability, cost vs scalability.
4. **Justify Choice:** Document why this solution was chosen.
5. **Review & Iterate:** Get feedback, adjust if needed.

**Example:**

> **Problem:** App frequently crashes on older devices with <2GB RAM.
>
> **Options:**
>
> - A) Reduce app size with lazy loading (6 weeks, risky)
> - B) Implement memory management improvements (2 weeks, immediate relief)
> - C) Drop support for devices <2GB (simple, loses users)
>
> **Decision:** Option B first, then A
>
> - Immediate mitigation (B) reduces crash reports.
> - Long-term solution (A) improves user experience.
> - Revisit C after metrics improve.

**2. Mentoring & Knowledge Sharing:**

**Effective Mentoring:**

- Lead by example (code quality, documentation).
- Pair programming on complex tasks.
- Code review with constructive feedback.
- Ask questions instead of giving answers (Socratic method).
- Create safe environment for questions.

**Code Review Best Practices:**

```
// BAD - Harsh feedback
"This code is garbage. Rewrite it."

// GOOD - Constructive feedback
"This logic can be simplified. Consider using a ternary operator here. 
See this example: [link]. Let me know if it works!"
```

**3. Handling Conflict:**

**Approach:**

- Listen without interrupting.
- Seek to understand, not to win.
- Focus on ideas, not personalities.
- Find common ground.
- Document disagreements and decisions.

**4. Stakeholder Communication:**

**Translating Technical to Business:**

- Avoid jargon. Use analogies.
- Focus on impact (time, cost, risk).
- Be transparent about trade-offs.

```
Technical: "We need to refactor the authentication module for maintainability."

Business-friendly: "We'll spend 2 weeks improving login reliability. 
This reduces crash reports by 30% and makes future updates faster."
```

**5. Agile & Scrum:**

**Agile Mindset:**

- Iterative development.
- Continuous feedback.
- Adapt to change.
- Value individuals over processes.

**Scrum Ceremonies:**

- **Sprint Planning:** Define sprint goals (1-2 weeks).
- **Daily Standup:** 15 min sync (what done, what next, blockers).
- **Sprint Review:** Demo completed work.
- **Retrospective:** What went well, what to improve.

**6. Handling Pressure & Deadlines:**

**When asked "Can you do this by Friday?":**

- Never commit immediately. Assess first.
- Provide options: "I can do X by Friday, Y by Monday, or both with help."
- Escalate if unrealistic.
- Communicate early if running behind.

**7. Common Behavioral Questions:**

| Question | Why Asked | Framework |
| --- | --- | --- |
| Tell me about a conflict you resolved. | Interpersonal skills | Situation → Action → Result |
| Describe a time you failed. | Accountability | Problem → Lesson → Growth |
| How do you handle pressure? | Stress management | Coping strategies → Outcomes |
| Tell about your proudest achievement. | Motivation | Challenge → Solution → Impact |
| How do you learn new technologies? | Growth mindset | Methods → Practice → Success |

**Answer Framework (STAR):**

- **Situation:** Context and setting.
- **Task:** Your role and goals.
- **Action:** Specific steps you took.
- **Result:** Outcome and learnings.

**Example:**

> **Q:** Tell me about a difficult technical decision you made.
>
> **A:**
>
> - **S:** Our app was growing, and monolithic architecture couldn't scale.
> - **T:** I was senior dev; needed to recommend next architecture.
> - **A:** I researched monoliths vs microservices, created comparison matrix, presented to team with pros/cons, proposed hybrid approach (monolith + async workers).
> - **R:** Implemented over 2 sprints, reduced API response time by 40%, improved developer velocity.

**8. Career Growth as Senior:**

- **Technical:** Master deep specialization, contribute to open source.
- **Leadership:** Mentor others, lead initiatives, make architecture decisions.
- **Communication:** Present at conferences, write articles, participate in communities.
- **Business:** Understand ROI, project planning, cost estimation.

**Best Practices:**

- Document decisions for future reference.
- Celebrate team wins, own failures personally.
- Create psychological safety for discussions.
- Balance quality with shipping.
- Always explain the "why", not just "how".

---

**Tip:** Practice coding, system design, and behavioral questions. Review recent Flutter releases and be ready to discuss trade-offs and real-world scenarios.

---

## 8. Git & Version Control

**Key Points:**

- Git is distributed; every clone is a full backup.
- Commit often with atomic, logical changes.
- Branches enable parallel development.
- Rebase creates linear history; merge preserves it.
- Pull requests enable code review and collaboration.

**Detailed Overview:**

**1. Core Concepts:**

**Commit Message Format (Conventional Commits):**

```
<type>(<scope>): <description>

feat: add user authentication
fix: handle null pointer in profile screen
docs: update README with setup instructions
```

**Types:** feat, fix, docs, style, refactor, perf, test, chore

**2. Key Workflows:**

**Creating and Pushing a Feature:**

```bash
git checkout -b feature/user-auth
# Make changes
git add .
git commit -m "feat: implement user authentication"
git push origin feature/user-auth
# Create PR on GitHub/GitLab for review
# After approval: merge and delete branch
```

**3. Merge vs Rebase:**

| Aspect | Merge | Rebase |
| --- | --- | --- |
| History | Preserves all branches | Linear, clean history |
| Safety | Safe, non-destructive | Rewrites history |
| Use Case | Shared branches | Personal/feature branches |
| Conflicts | Merge commit | Applied one-by-one |

**4. Advanced Operations:**

**Cherry-pick (Apply specific commit):**

```bash
git cherry-pick abc123  # Apply commit abc123 to current branch
```

**Revert (Safe undo):**

```bash
git revert abc123  # Creates new commit that undoes abc123
```

**Reset (Dangerous):**

```bash
git reset --soft HEAD~1   # Undo, keep changes staged
git reset --hard HEAD~1   # Undo, discard changes (DESTRUCTIVE)
```

**Interactive Rebase (Squash commits):**

```bash
git rebase -i HEAD~3  # Edit last 3 commits
# In editor: pick, squash, reword, fixup
```

**5. Resolving Merge Conflicts:**

```
File shows:
<<<<<< HEAD
    return user.name;
======
    return getUserName();
>>>>>> feature-branch

Choose one or combine, then:
git add <file>
git commit -m "Resolve merge conflict"
```

**6. Stashing (Temporary Save):**

```bash
git stash              # Save uncommitted changes
git stash list         # Show all stashes
git stash pop          # Apply and remove latest
git stash apply stash@{0}  # Apply specific stash
```

**7. Recovery (Detached HEAD, Deleted Branches):**

```bash
git reflog             # Shows all HEAD movements
git checkout -b recovery <commit-hash>  # Recover from detached state
```

**8. Best Practices:**

- Pull before push.
- Never force push to shared branches.
- Write meaningful commit messages.
- Commit logically, not randomly.
- Keep branches short-lived (ideally < 1 week).
- Tag releases: `git tag -a v1.0.0 -m "Release v1.0.0"`

**9. Git Aliases for Efficiency:**

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.visual 'log --graph --oneline --all'
```

**10. Common Scenarios:**

**Committed to wrong branch:**

```bash
git reset --soft HEAD~1
git stash
git checkout correct-branch
git stash pop
git commit -m "message"
```

**Need to undo pushed commit:**

```bash
git revert <commit-hash>  # Safe
git push
# NEVER use git reset --hard if already pushed
```

**Wrote message with typo:**

```bash
git commit --amend --no-edit  # Fix last commit
```

---

## Quick Reference & Revision Checklist

**Before Every Interview:**

- [ ] Review Flutter widget lifecycle
- [ ] Practice 5 DSA problems (arrays, trees, graphs)
- [ ] Recall state management patterns
- [ ] Review your past projects and trade-offs
- [ ] Practice one system design scenario
- [ ] Prepare 2-3 behavioral stories (STAR format)
- [ ] Test your Git workflow

**Day Before Interview:**

- [ ] Don't cram; rest well.
- [ ] Review key points only.
- [ ] Ensure laptop/environment set up.
- [ ] Prepare questions to ask interviewers.

**During Interview:**

- [ ] Think aloud; don't stay silent.
- [ ] Ask clarifying questions before jumping into solutions.
- [ ] Trade-offs are crucial; discuss them.
- [ ] Be honest about what you don't know.
- [ ] Ask follow-up questions at the end.

---

**Final Tips:**

1. **Master the fundamentals:** Strong fundamentals > surface-level breadth.
2. **Project experience matters:** Be ready to deep-dive into your projects.
3. **Communication is key:** Senior role = mentoring and explaining.
4. **Practice, practice, practice:** Use LeetCode, HackerRank, CodeSignal.
5. **Stay updated:** Follow Flutter announcements, read Medium articles, attend conferences.
6. **Help others:** Teaching cements your own learning.

---

Good luck with your interview! Remember: being a senior engineer is not just about technical skills—it's about problem-solving, communication, and making good decisions under uncertainty.
