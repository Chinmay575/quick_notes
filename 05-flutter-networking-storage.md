# 05 — Flutter Networking & Storage

---

## Q1. How do you make HTTP requests in Flutter?

**Answer:**

```dart
// Using http package
import 'package:http/http.dart' as http;

Future<List<User>> fetchUsers() async {
  final response = await http.get(
    Uri.parse('https://api.example.com/users'),
    headers: {'Authorization': 'Bearer $token'},
  );

  if (response.statusCode == 200) {
    final List<dynamic> data = jsonDecode(response.body);
    return data.map((json) => User.fromJson(json)).toList();
  } else {
    throw HttpException('Failed: ${response.statusCode}');
  }
}

// POST
final response = await http.post(
  Uri.parse('https://api.example.com/users'),
  headers: {'Content-Type': 'application/json'},
  body: jsonEncode({'name': 'Alice', 'email': 'alice@test.com'}),
);
```

---

## Q2. What is Dio and why use it over http package?

**Answer:**

```dart
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: Duration(seconds: 10),
  receiveTimeout: Duration(seconds: 10),
  headers: {'Content-Type': 'application/json'},
));

// Interceptors
dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
    options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  },
  onResponse: (response, handler) {
    handler.next(response);
  },
  onError: (error, handler) {
    if (error.response?.statusCode == 401) {
      // Refresh token logic
    }
    handler.next(error);
  },
));

// GET with query parameters
final response = await dio.get('/users', queryParameters: {'page': 1});

// POST
await dio.post('/users', data: {'name': 'Alice'});

// File upload
final formData = FormData.fromMap({
  'file': await MultipartFile.fromFile('/path/to/file.jpg'),
  'name': 'profile',
});
await dio.post('/upload', data: formData);

// Download with progress
await dio.download(
  'https://example.com/file.zip',
  '/path/to/save.zip',
  onReceiveProgress: (received, total) {
    print('${(received / total * 100).toStringAsFixed(0)}%');
  },
);

// Cancel requests
final cancelToken = CancelToken();
dio.get('/slow-api', cancelToken: cancelToken);
cancelToken.cancel('User cancelled');
```

**Dio advantages:** Interceptors, cancel tokens, file upload/download progress, FormData, global config, retry.

---

## Q3. How do you handle JSON serialization/deserialization?

**Answer:**

```dart
// Manual
class User {
  final int id;
  final String name;
  final String email;

  User({required this.id, required this.name, required this.email});

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
      email: json['email'] as String,
    );
  }

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
  };
}

// With json_serializable (code generation)
@JsonSerializable()
class User {
  final int id;
  final String name;
  @JsonKey(name: 'email_address')  // custom JSON key
  final String email;
  @JsonKey(defaultValue: false)
  final bool isActive;

  User({required this.id, required this.name, required this.email, this.isActive = false});

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);
}

// Run: dart run build_runner build

// With freezed (immutable + code generation)
@freezed
class User with _$User {
  const factory User({
    required int id,
    required String name,
    @Default(false) bool isActive,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

---

## Q4. What is REST API vs GraphQL? How do you use GraphQL in Flutter?

**Answer:**

| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoints | Multiple (`/users`, `/posts`) | Single (`/graphql`) |
| Data fetching | Fixed response structure | Client specifies exact fields |
| Over/Under-fetching | Common problem | Solved by design |
| Caching | HTTP caching (simple) | Normalized cache (complex) |
| Versioning | URL versioning (`/v1/users`) | No versioning needed |

```dart
// Using graphql_flutter
final client = GraphQLClient(
  link: HttpLink('https://api.example.com/graphql'),
  cache: GraphQLCache(store: HiveStore()),
);

// Query
final result = await client.query(QueryOptions(
  document: gql(r'''
    query GetUser($id: ID!) {
      user(id: $id) {
        name
        email
        posts { title }
      }
    }
  '''),
  variables: {'id': '1'},
));

// Mutation
await client.mutate(MutationOptions(
  document: gql(r'''
    mutation CreateUser($input: CreateUserInput!) {
      createUser(input: $input) { id name }
    }
  '''),
  variables: {'input': {'name': 'Alice', 'email': 'alice@test.com'}},
));
```

---

## Q5. How do you implement offline-first / caching strategies?

**Answer:**

```dart
// Strategy: Cache-first, then network
class UserRepository {
  final ApiClient api;
  final LocalCache cache;

  Future<List<User>> getUsers() async {
    // 1. Return cached data immediately
    final cached = await cache.getUsers();
    if (cached != null) return cached;

    // 2. Fetch from network
    try {
      final users = await api.fetchUsers();
      await cache.saveUsers(users); // update cache
      return users;
    } catch (e) {
      // 3. Return stale cache if network fails
      final stale = await cache.getUsers(allowStale: true);
      if (stale != null) return stale;
      rethrow;
    }
  }
}

// Using connectivity_plus to detect network
final subscription = Connectivity().onConnectivityChanged.listen((result) {
  if (result != ConnectivityResult.none) {
    syncPendingChanges(); // sync when back online
  }
});
```

---

## Q6. Explain SharedPreferences in Flutter.

**Answer:**

```dart
// Simple key-value storage (NOT for sensitive data)
final prefs = await SharedPreferences.getInstance();

// Write
await prefs.setString('username', 'Alice');
await prefs.setInt('loginCount', 5);
await prefs.setBool('darkMode', true);
await prefs.setDouble('rating', 4.5);
await prefs.setStringList('tags', ['flutter', 'dart']);

// Read
String? username = prefs.getString('username');
int loginCount = prefs.getInt('loginCount') ?? 0;
bool darkMode = prefs.getBool('darkMode') ?? false;

// Remove
await prefs.remove('username');

// Clear all
await prefs.clear();
```

**Use cases:** Theme preference, onboarding flag, locale selection, feature flags.
**NOT for:** Sensitive data (passwords, tokens) — use `flutter_secure_storage`.

---

## Q7. Explain SQLite (sqflite) in Flutter.

**Answer:**

```dart
class DatabaseHelper {
  static Database? _database;

  Future<Database> get database async {
    _database ??= await _initDB();
    return _database!;
  }

  Future<Database> _initDB() async {
    final path = join(await getDatabasesPath(), 'app.db');
    return openDatabase(
      path,
      version: 2,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
          )
        ''');
      },
      onUpgrade: (db, oldVersion, newVersion) async {
        if (oldVersion < 2) {
          await db.execute('ALTER TABLE users ADD COLUMN avatar TEXT');
        }
      },
    );
  }

  // CRUD
  Future<int> insertUser(User user) async {
    final db = await database;
    return db.insert('users', user.toMap(), conflictAlgorithm: ConflictAlgorithm.replace);
  }

  Future<List<User>> getUsers() async {
    final db = await database;
    final maps = await db.query('users', orderBy: 'name ASC');
    return maps.map((m) => User.fromMap(m)).toList();
  }

  Future<int> updateUser(User user) async {
    final db = await database;
    return db.update('users', user.toMap(), where: 'id = ?', whereArgs: [user.id]);
  }

  Future<int> deleteUser(int id) async {
    final db = await database;
    return db.delete('users', where: 'id = ?', whereArgs: [id]);
  }
}
```

---

## Q8. What is Hive and when to use it vs SQLite?

**Answer:**

```dart
// Initialize
await Hive.initFlutter();
Hive.registerAdapter(UserAdapter());

// Open box
var box = await Hive.openBox<User>('users');

// CRUD
await box.put('alice', User(name: 'Alice', age: 30));
User? user = box.get('alice');
await box.delete('alice');
await box.clear();

// Auto-increment keys
await box.add(User(name: 'Bob', age: 25)); // key = 0, 1, 2...

// TypeAdapter (or use hive_generator)
@HiveType(typeId: 0)
class User extends HiveObject {
  @HiveField(0)
  String name;
  @HiveField(1)
  int age;
  User({required this.name, required this.age});
}
```

| Feature | Hive | SQLite |
|---------|------|--------|
| Type | NoSQL (key-value) | Relational (SQL) |
| Speed | Very fast | Fast |
| Queries | Limited | Full SQL |
| Schema | Flexible | Structured |
| Relations | Manual | JOIN, foreign keys |
| Best for | Settings, cache, simple data | Complex data with relationships |
| Web support | Yes | No (use drift) |

---

## Q9. How do you store sensitive data securely?

**Answer:**

```dart
// flutter_secure_storage — uses Keychain (iOS) / EncryptedSharedPreferences (Android)
final storage = FlutterSecureStorage();

// Write
await storage.write(key: 'auth_token', value: 'eyJhbGciOi...');
await storage.write(key: 'refresh_token', value: 'dGhpcyBpcyBh...');

// Read
String? token = await storage.read(key: 'auth_token');

// Delete
await storage.delete(key: 'auth_token');

// Delete all
await storage.deleteAll();

// Check existence
bool exists = await storage.containsKey(key: 'auth_token');

// Android options
final androidOptions = AndroidOptions(encryptedSharedPreferences: true);
final storage = FlutterSecureStorage(aOptions: androidOptions);
```

**Best practices:**
- Store tokens in secure storage, NOT SharedPreferences
- Don't store sensitive data in plain text files
- Use biometric authentication for extra security
- Clear sensitive data on logout

---

## Q10. What is Drift (formerly Moor)?

**Answer:**

```dart
// Type-safe, reactive SQLite wrapper with code generation
@DriftDatabase(tables: [Users, Tasks])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  // Type-safe queries
  Future<List<User>> getAllUsers() => select(users).get();

  Stream<List<User>> watchAllUsers() => select(users).watch(); // reactive!

  Future<int> insertUser(UsersCompanion user) => into(users).insert(user);

  Future<List<User>> searchUsers(String query) {
    return (select(users)..where((u) => u.name.like('%$query%'))).get();
  }
}

// Table definition
class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 100)();
  TextColumn get email => text().unique()();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}
```

**Advantages:** Compile-time SQL verification, reactive streams, type safety, migration support, works on all platforms.

---

## Q11. How do you implement WebSocket connections in Flutter?

**Answer:**

```dart
// Using web_socket_channel
import 'package:web_socket_channel/web_socket_channel.dart';

class WebSocketService {
  late WebSocketChannel _channel;

  void connect() {
    _channel = WebSocketChannel.connect(Uri.parse('wss://echo.websocket.org'));

    // Listen for messages
    _channel.stream.listen(
      (message) => print('Received: $message'),
      onError: (error) => print('Error: $error'),
      onDone: () {
        print('Connection closed');
        _reconnect(); // auto-reconnect
      },
    );
  }

  void send(String message) {
    _channel.sink.add(message);
  }

  void disconnect() {
    _channel.sink.close();
  }

  void _reconnect() {
    Future.delayed(Duration(seconds: 5), connect);
  }
}

// In UI with StreamBuilder
StreamBuilder(
  stream: webSocketService.channel.stream,
  builder: (context, snapshot) {
    if (snapshot.hasData) return Text('${snapshot.data}');
    return CircularProgressIndicator();
  },
)
```

---

## Q12. How do you handle file operations (read/write/pick)?

**Answer:**

```dart
// File picker
import 'package:file_picker/file_picker.dart';

FilePickerResult? result = await FilePicker.platform.pickFiles(
  type: FileType.custom,
  allowedExtensions: ['jpg', 'png', 'pdf'],
  allowMultiple: true,
);
if (result != null) {
  for (var file in result.files) {
    print('Name: ${file.name}, Size: ${file.size}, Path: ${file.path}');
  }
}

// Read/write to app directory
import 'package:path_provider/path_provider.dart';

Future<File> _getLocalFile(String filename) async {
  final dir = await getApplicationDocumentsDirectory();
  return File('${dir.path}/$filename');
}

// Write
final file = await _getLocalFile('data.json');
await file.writeAsString(jsonEncode(data));

// Read
final contents = await file.readAsString();
final data = jsonDecode(contents);

// Image picker
final picker = ImagePicker();
final XFile? image = await picker.pickImage(
  source: ImageSource.camera, // or ImageSource.gallery
  maxWidth: 800,
  imageQuality: 80,
);
```
