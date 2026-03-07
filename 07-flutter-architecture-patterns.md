# 07 — Flutter Architecture & Design Patterns

---

## Q1. Explain Clean Architecture in Flutter.

**Answer:**

```
lib/
├── core/               # Shared utilities, errors, constants
│   ├── error/          # Failures, exceptions
│   ├── usecases/       # Base use case class
│   └── network/        # Network info
├── features/
│   └── auth/
│       ├── data/       # Layer 3: Data (outermost)
│       │   ├── datasources/   # Remote (API) and Local (DB)
│       │   ├── models/        # Data transfer objects (JSON)
│       │   └── repositories/  # Implements domain repo interface
│       ├── domain/     # Layer 1: Domain (innermost, no dependencies)
│       │   ├── entities/      # Business objects
│       │   ├── repositories/  # Abstract repository interface
│       │   └── usecases/      # Business rules
│       └── presentation/ # Layer 2: Presentation
│           ├── bloc/          # State management
│           ├── pages/         # Screens
│           └── widgets/       # UI components
```

**Dependency Rule:** Dependencies point INWARD. Domain knows nothing about Data or Presentation.

```dart
// Domain layer — Entity
class User {
  final int id;
  final String name;
  User({required this.id, required this.name});
}

// Domain layer — Repository interface
abstract class UserRepository {
  Future<Either<Failure, List<User>>> getUsers();
}

// Domain layer — Use Case
class GetUsers {
  final UserRepository repository;
  GetUsers(this.repository);
  Future<Either<Failure, List<User>>> call() => repository.getUsers();
}

// Data layer — Model (extends Entity)
class UserModel extends User {
  UserModel({required super.id, required super.name});
  factory UserModel.fromJson(Map<String, dynamic> json) =>
    UserModel(id: json['id'], name: json['name']);
}

// Data layer — Repository implementation
class UserRepositoryImpl implements UserRepository {
  final RemoteDataSource remote;
  final LocalDataSource local;
  final NetworkInfo network;

  @override
  Future<Either<Failure, List<User>>> getUsers() async {
    if (await network.isConnected) {
      try {
        final users = await remote.getUsers();
        await local.cacheUsers(users);
        return Right(users);
      } catch (e) {
        return Left(ServerFailure());
      }
    } else {
      final cached = await local.getCachedUsers();
      return Right(cached);
    }
  }
}
```

---

## Q2. Explain MVVM pattern in Flutter.

**Answer:**

```
View (Widget) ←→ ViewModel (ChangeNotifier/Bloc) ←→ Model (Data)
```

```dart
// Model
class User {
  final String name;
  final String email;
  User({required this.name, required this.email});
}

// ViewModel
class UserViewModel extends ChangeNotifier {
  final UserRepository _repository;
  List<User> _users = [];
  bool _isLoading = false;
  String? _error;

  List<User> get users => _users;
  bool get isLoading => _isLoading;
  String? get error => _error;

  UserViewModel(this._repository);

  Future<void> loadUsers() async {
    _isLoading = true;
    _error = null;
    notifyListeners();
    try {
      _users = await _repository.getUsers();
    } catch (e) {
      _error = e.toString();
    }
    _isLoading = false;
    notifyListeners();
  }
}

// View
class UserListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer<UserViewModel>(
      builder: (_, vm, __) {
        if (vm.isLoading) return CircularProgressIndicator();
        if (vm.error != null) return Text('Error: ${vm.error}');
        return ListView.builder(
          itemCount: vm.users.length,
          itemBuilder: (_, i) => ListTile(title: Text(vm.users[i].name)),
        );
      },
    );
  }
}
```

---

## Q3. Explain SOLID principles with Flutter examples.

**Answer:**

**S — Single Responsibility:**
```dart
// ❌ Widget does fetching + UI + validation
// ✅ Separate into: ApiService, UserValidator, UserListWidget
```

**O — Open/Closed (open for extension, closed for modification):**
```dart
abstract class PaymentMethod {
  Future<bool> pay(double amount);
}
class CreditCard extends PaymentMethod { ... }
class GooglePay extends PaymentMethod { ... }
// Add new payment methods without modifying existing code
```

**L — Liskov Substitution:**
```dart
// Any subtype should be usable wherever the parent type is expected
void processPayment(PaymentMethod method) => method.pay(100);
processPayment(CreditCard());  // works
processPayment(GooglePay());   // works
```

**I — Interface Segregation:**
```dart
// ❌ One fat interface
abstract class Worker { void code(); void design(); void manage(); }

// ✅ Small, focused interfaces
abstract class Coder { void code(); }
abstract class Designer { void design(); }
class Developer implements Coder { void code() { ... } }
```

**D — Dependency Inversion:**
```dart
// ❌ High-level depends on low-level
class UserBloc { final SqlDatabase db = SqlDatabase(); }

// ✅ Both depend on abstraction
class UserBloc { final UserRepository repo; UserBloc(this.repo); }
abstract class UserRepository { Future<List<User>> getUsers(); }
class SqlUserRepository implements UserRepository { ... }
class ApiUserRepository implements UserRepository { ... }
```

---

## Q4. How do you implement Dependency Injection in Flutter?

**Answer:**

```dart
// 1. get_it (Service Locator)
final getIt = GetIt.instance;

void setup() {
  // Singleton
  getIt.registerSingleton<ApiClient>(ApiClient());

  // Lazy singleton
  getIt.registerLazySingleton<Database>(() => Database());

  // Factory (new instance each time)
  getIt.registerFactory<UserRepository>(() =>
    UserRepositoryImpl(getIt<ApiClient>()));

  // With dispose
  getIt.registerLazySingleton<UserBloc>(
    () => UserBloc(getIt<UserRepository>()),
    dispose: (bloc) => bloc.close(),
  );
}

// Usage
final repo = getIt<UserRepository>();

// 2. injectable (code gen for get_it)
@injectable
class UserRepository {
  final ApiClient client;
  UserRepository(this.client);
}

@module
abstract class AppModule {
  @lazySingleton
  Dio get dio => Dio();
}

// 3. Provider (built-in DI)
MultiProvider(
  providers: [
    Provider(create: (_) => ApiClient()),
    ProxyProvider<ApiClient, UserRepository>(
      update: (_, api, __) => UserRepository(api),
    ),
  ],
  child: MyApp(),
)
```

---

## Q5. What is the Repository pattern?

**Answer:**
The Repository pattern abstracts data sources behind a single interface, so business logic doesn't know if data comes from API, cache, or database.

```dart
// Abstract interface
abstract class UserRepository {
  Future<List<User>> getUsers();
  Future<User> getUserById(int id);
  Future<void> saveUser(User user);
}

// Implementation deciding data source
class UserRepositoryImpl implements UserRepository {
  final RemoteDataSource _remote;
  final LocalDataSource _local;
  final ConnectivityChecker _connectivity;

  @override
  Future<List<User>> getUsers() async {
    if (await _connectivity.isOnline) {
      final users = await _remote.fetchUsers();
      await _local.cacheUsers(users);
      return users;
    }
    return _local.getCachedUsers();
  }
}
```

**Benefits:** Testable (mock the interface), swappable data sources, single source of truth, offline support.

---

## Q6. Explain the Observer pattern in Flutter.

**Answer:**
Observer pattern is EVERYWHERE in Flutter:

```dart
// 1. ChangeNotifier + Listener
class Counter extends ChangeNotifier {
  int value = 0;
  void increment() { value++; notifyListeners(); }
}
final counter = Counter();
counter.addListener(() => print('Changed: ${counter.value}'));

// 2. Streams
final controller = StreamController<int>.broadcast();
controller.stream.listen((data) => print('Observer 1: $data'));
controller.stream.listen((data) => print('Observer 2: $data'));
controller.sink.add(42); // both observers notified

// 3. ValueNotifier
final name = ValueNotifier<String>('Alice');
name.addListener(() => print(name.value));

// 4. BLoC — Events are observed, States are emitted
```

---

## Q7. What are some common design patterns used in Flutter?

**Answer:**

| Pattern | Example in Flutter |
|---------|-------------------|
| **Singleton** | `GetIt`, `SharedPreferences.getInstance()` |
| **Factory** | `factory` constructors, `Widget.fromJson()` |
| **Builder** | `FutureBuilder`, `StreamBuilder`, `LayoutBuilder`, `BlocBuilder` |
| **Observer** | `ChangeNotifier`, `Stream`, `BLoC` |
| **Strategy** | Different `PaymentMethod` implementations |
| **Composite** | Widget tree (widgets containing widgets) |
| **Decorator** | `Padding`, `Container` wrapping child widgets |
| **Proxy** | Repository pattern (proxy to data sources) |
| **Command** | BLoC events, `VoidCallback` |
| **State** | `StatefulWidget`, state management solutions |
| **Adapter** | Data Models adapting API responses to domain entities |
| **Template Method** | `StatefulWidget` lifecycle methods |

---

## Q8. How do you structure a large Flutter project?

**Answer:**

```
lib/
├── app/                    # App-level config
│   ├── app.dart           # MaterialApp
│   ├── routes.dart        # Route definitions
│   └── theme.dart         # ThemeData
├── core/                   # Shared across features
│   ├── constants/
│   ├── errors/
│   ├── extensions/
│   ├── network/
│   ├── utils/
│   └── widgets/           # Reusable widgets
├── features/               # Feature modules
│   ├── auth/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── home/
│   ├── profile/
│   └── settings/
├── l10n/                   # Localization
├── di/                     # Dependency injection setup
│   └── injection.dart
└── main.dart
```

**Principles:**
- Feature-first organization (not layer-first)
- Each feature is self-contained
- Shared code goes in `core/`
- Avoid circular dependencies between features

---

## Q9. What is the Either type and functional error handling?

**Answer:**

```dart
// Using dartz or fpdart package
import 'package:dartz/dartz.dart';

abstract class Failure {
  final String message;
  Failure(this.message);
}
class ServerFailure extends Failure { ServerFailure(super.message); }
class CacheFailure extends Failure { CacheFailure(super.message); }

// Returns Either<Failure, Success>
Future<Either<Failure, User>> getUser(int id) async {
  try {
    final response = await api.get('/users/$id');
    return Right(User.fromJson(response.data)); // success
  } on DioException catch (e) {
    return Left(ServerFailure(e.message ?? 'Unknown')); // failure
  }
}

// Usage — fold to handle both cases
final result = await getUser(1);
result.fold(
  (failure) => showError(failure.message), // Left (error)
  (user) => showUser(user),                // Right (success)
);
```

**Benefits:** No try-catch in UI, explicit error handling, type-safe, composable.
