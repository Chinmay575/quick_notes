# 06 — Flutter Testing & Debugging

---

## Q1. What are the types of testing in Flutter?

**Answer:**

| Type | Speed | Scope | Tools |
|------|-------|-------|-------|
| **Unit tests** | Fast | Single function/class | `test`, `mockito` |
| **Widget tests** | Medium | Single widget | `flutter_test`, `WidgetTester` |
| **Integration tests** | Slow | Full app flow | `integration_test`, `patrol` |
| **Golden tests** | Medium | Visual snapshot comparison | `flutter_test`, `golden_toolkit` |

**Test pyramid:** Many unit tests → fewer widget tests → fewest integration tests.

---

## Q2. How do you write unit tests in Flutter?

**Answer:**

```dart
// test/calculator_test.dart
import 'package:test/test.dart';

class Calculator {
  double add(double a, double b) => a + b;
  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('Cannot divide by zero');
    return a / b;
  }
}

void main() {
  late Calculator calculator;

  setUp(() {
    calculator = Calculator();
  });

  group('Calculator', () {
    test('adds two numbers', () {
      expect(calculator.add(2, 3), equals(5));
    });

    test('divides two numbers', () {
      expect(calculator.divide(10, 2), equals(5));
    });

    test('throws on divide by zero', () {
      expect(() => calculator.divide(10, 0), throwsArgumentError);
    });
  });
}

// Run: flutter test test/calculator_test.dart
```

---

## Q3. How do you mock dependencies in tests?

**Answer:**

```dart
// Using mockito + build_runner
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';

@GenerateMocks([UserRepository, ApiClient])
void main() {
  late MockUserRepository mockRepo;
  late UserBloc bloc;

  setUp(() {
    mockRepo = MockUserRepository();
    bloc = UserBloc(mockRepo);
  });

  test('fetches users successfully', () async {
    // Arrange
    when(mockRepo.getUsers()).thenAnswer(
      (_) async => [User(id: 1, name: 'Alice')],
    );

    // Act
    final users = await bloc.loadUsers();

    // Assert
    expect(users.length, equals(1));
    expect(users.first.name, equals('Alice'));
    verify(mockRepo.getUsers()).called(1);
    verifyNoMoreInteractions(mockRepo);
  });

  test('handles error', () async {
    when(mockRepo.getUsers()).thenThrow(Exception('Network error'));
    expect(() => bloc.loadUsers(), throwsException);
  });
}

// Using mocktail (no code generation)
import 'package:mocktail/mocktail.dart';

class MockUserRepository extends Mock implements UserRepository {}

// Same test but with mocktail syntax
when(() => mockRepo.getUsers()).thenAnswer((_) async => []);
verify(() => mockRepo.getUsers()).called(1);
```

---

## Q4. How do you write widget tests?

**Answer:**

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('Counter increments', (WidgetTester tester) async {
    // Build widget
    await tester.pumpWidget(MaterialApp(home: CounterPage()));

    // Verify initial state
    expect(find.text('0'), findsOneWidget);
    expect(find.text('1'), findsNothing);

    // Tap button
    await tester.tap(find.byIcon(Icons.add));
    await tester.pump(); // rebuild after setState

    // Verify new state
    expect(find.text('1'), findsOneWidget);
  });

  testWidgets('shows loading then data', (tester) async {
    await tester.pumpWidget(MaterialApp(home: UserListPage()));

    // Loading state
    expect(find.byType(CircularProgressIndicator), findsOneWidget);

    // Wait for async operations to complete
    await tester.pumpAndSettle();

    // Data loaded
    expect(find.byType(CircularProgressIndicator), findsNothing);
    expect(find.byType(ListTile), findsWidgets);
  });

  testWidgets('form validation', (tester) async {
    await tester.pumpWidget(MaterialApp(home: LoginPage()));

    // Enter text
    await tester.enterText(find.byKey(Key('email_field')), 'invalid');
    await tester.tap(find.text('Submit'));
    await tester.pump();

    // Verify error
    expect(find.text('Invalid email'), findsOneWidget);
  });
}
```

**Finders:** `find.text()`, `find.byType()`, `find.byKey()`, `find.byIcon()`, `find.byWidget()`, `find.descendant()`, `find.ancestor()`

**Matchers:** `findsOneWidget`, `findsNothing`, `findsWidgets`, `findsNWidgets(n)`, `findsAtLeast(n)`

---

## Q5. How do you write integration tests?

**Answer:**

```dart
// integration_test/app_test.dart
import 'package:integration_test/integration_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('full login flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    // Login screen
    await tester.enterText(find.byKey(Key('email')), 'test@test.com');
    await tester.enterText(find.byKey(Key('password')), 'password123');
    await tester.tap(find.text('Login'));
    await tester.pumpAndSettle();

    // Verify navigation to home
    expect(find.text('Welcome'), findsOneWidget);

    // Navigate
    await tester.tap(find.byIcon(Icons.settings));
    await tester.pumpAndSettle();
    expect(find.text('Settings'), findsOneWidget);
  });
}

// Run: flutter test integration_test/app_test.dart
```

---

## Q6. What are Golden tests (snapshot testing)?

**Answer:**

```dart
testWidgets('renders correctly', (tester) async {
  await tester.pumpWidget(MaterialApp(
    home: Scaffold(body: MyCustomWidget()),
  ));

  // Generate golden file (first run)
  await expectLater(
    find.byType(MyCustomWidget),
    matchesGoldenFile('goldens/my_widget.png'),
  );
});

// Generate: flutter test --update-goldens
// Verify: flutter test (fails if visual difference detected)
```

**Use cases:** Design consistency, catch unintended UI changes, component library validation.

---

## Q7. How do you test BLoC/Cubit?

**Answer:**

```dart
// Using bloc_test
import 'package:bloc_test/bloc_test.dart';

void main() {
  group('CounterBloc', () {
    blocTest<CounterBloc, int>(
      'emits [1] when IncrementEvent is added',
      build: () => CounterBloc(),
      act: (bloc) => bloc.add(IncrementEvent()),
      expect: () => [1],
    );

    blocTest<CounterBloc, int>(
      'emits [1, 2, 3] for three increments',
      build: () => CounterBloc(),
      act: (bloc) {
        bloc.add(IncrementEvent());
        bloc.add(IncrementEvent());
        bloc.add(IncrementEvent());
      },
      expect: () => [1, 2, 3],
    );

    blocTest<UserBloc, UserState>(
      'fetches users with mock repo',
      setUp: () {
        when(() => mockRepo.getUsers()).thenAnswer((_) async => [testUser]);
      },
      build: () => UserBloc(mockRepo),
      act: (bloc) => bloc.add(FetchUsersEvent()),
      expect: () => [
        UserLoading(),
        UserLoaded([testUser]),
      ],
      verify: (_) {
        verify(() => mockRepo.getUsers()).called(1);
      },
    );
  });
}
```

---

## Q8. What is Flutter DevTools?

**Answer:**
Flutter DevTools is a suite of performance and debugging tools:

| Tool | Purpose |
|------|---------|
| **Widget Inspector** | Explore widget tree, view properties, constraints |
| **Performance** | Timeline, frame rendering, jank detection |
| **CPU Profiler** | Function-level CPU usage |
| **Memory** | Heap snapshots, allocation tracking, leak detection |
| **Network** | HTTP request/response inspector |
| **Logging** | View `log()` output and errors |
| **App Size** | Analyze APK/IPA size breakdown |

```bash
# Launch DevTools
flutter pub global activate devtools
dart devtools

# Or from VS Code: Ctrl+Shift+P → "Dart: Open DevTools"
```

---

## Q9. How do you debug layout issues?

**Answer:**

```dart
// 1. Debug paint — shows constraints and boundaries
import 'package:flutter/rendering.dart';
debugPaintSizeEnabled = true;
debugPaintBaselinesEnabled = true;
debugPaintPointersEnabled = true;

// 2. Repaint rainbow — shows which widgets are repainting
debugRepaintRainbowEnabled = true;

// 3. Widget Inspector in DevTools
// Shows the entire widget tree, Element tree, and RenderObject tree

// 4. Layout Explorer in DevTools
// Visual flex/constraint debugging

// 5. Common layout errors and fixes
// "RenderFlex overflowed" → Wrap in Expanded, SingleChildScrollView, or use Flexible
// "Unbounded constraints" → Give explicit size or wrap in Expanded
// "A RenderFlex overflowed by X pixels" → Check mainAxisSize, add scrolling
```

---

## Q10. How do you handle errors globally in Flutter?

**Answer:**

```dart
void main() {
  // Catch Flutter framework errors
  FlutterError.onError = (FlutterErrorDetails details) {
    FlutterError.presentError(details); // default behavior
    // Send to crash reporting (Firebase Crashlytics, Sentry)
    FirebaseCrashlytics.instance.recordFlutterError(details);
  };

  // Catch async errors not caught by Flutter
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack);
    return true; // handled
  };

  // Catch errors in zones
  runZonedGuarded(() {
    runApp(MyApp());
  }, (error, stackTrace) {
    // All uncaught errors land here
    log('Uncaught error', error: error, stackTrace: stackTrace);
  });
}

// Custom error widget (instead of red error screen)
ErrorWidget.builder = (FlutterErrorDetails details) {
  return Center(
    child: Text('Something went wrong', style: TextStyle(color: Colors.red)),
  );
};
```

---

## Q11. What is code coverage and how do you measure it?

**Answer:**

```bash
# Generate coverage
flutter test --coverage

# View report (requires lcov)
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html

# Specific file
flutter test --coverage test/unit/
```

**Coverage types:**
- **Line coverage** — % of lines executed
- **Branch coverage** — % of conditional branches taken
- **Function coverage** — % of functions called

**Good target:** 70-80% for business logic, 50-60% for UI code.

---

## Q12. How do you test async code?

**Answer:**

```dart
test('fetches data asynchronously', () async {
  final service = UserService(mockApi);
  final users = await service.getUsers();
  expect(users, isNotEmpty);
});

// With fake async (controlling time)
import 'package:fake_async/fake_async.dart';

test('debounce search', () {
  fakeAsync((async) {
    final controller = SearchController();
    controller.search('flutter');

    // Advance time by debounce duration
    async.elapse(Duration(milliseconds: 300));

    expect(controller.results, isNotEmpty);
  });
});

// Testing streams
test('emits values in order', () {
  expect(
    counterStream(),
    emitsInOrder([0, 1, 2, 3, emitsDone]),
  );
});

test('emits error', () {
  expect(
    errorStream(),
    emitsError(isA<NetworkException>()),
  );
});
```
