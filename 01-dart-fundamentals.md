# 01 — Dart Fundamentals

---

## Q1. What is Dart? Why does Flutter use Dart?

**Answer:**
Dart is an object-oriented, client-optimized language developed by Google. Flutter uses Dart because:

- **AOT (Ahead-of-Time) compilation** — produces fast native ARM code for release builds.
- **JIT (Just-in-Time) compilation** — enables hot reload during development.
- **Single-threaded event loop** — no locking issues; UI is smooth.
- **Strong typing with type inference** — catches errors at compile time.
- **Dart compiles to native code** for mobile, JavaScript for web, and machine code for desktop.

---

## Q2. What is the difference between `var`, `dynamic`, `final`, `const`, and `late` in Dart?

**Answer:**

| Keyword | Type Inference | Reassignable | Compile-time constant | Initialization |
|---------|---------------|-------------|----------------------|----------------|
| `var` | Yes (locked after first assignment type) | Yes | No | At declaration |
| `dynamic` | No (any type, checked at runtime) | Yes | No | At declaration |
| `final` | Yes | No (single assignment) | No (runtime constant) | At declaration or constructor |
| `const` | Yes | No | Yes (compile-time constant) | Must be compile-time value |
| `late` | Yes | Yes (unless `late final`) | No | Deferred (lazy initialization) |

```dart
var name = 'Alice';        // String, inferred, reassignable
dynamic anything = 42;     // can be reassigned to any type
final city = 'NYC';        // runtime constant, assigned once
const pi = 3.14159;        // compile-time constant
late String description;   // initialized later, before first use
```

---

## Q3. Explain Null Safety in Dart.

**Answer:**
Dart's sound null safety (since Dart 2.12) ensures that variables cannot contain `null` unless explicitly declared nullable.

```dart
String name = 'Alice';   // non-nullable — CANNOT be null
String? name2;           // nullable — CAN be null

// Null-aware operators:
name2?.length;         // safe access — returns null if name2 is null
name2 ?? 'default';    // if-null — returns 'default' if name2 is null
name2 ??= 'fallback';  // assign if null
name2!.length;         // null assertion — throws if null (use carefully)
```

**Key concepts:**
- **Non-nullable by default** — every type is non-nullable unless you add `?`.
- **Flow analysis** — the compiler tracks null checks and promotes types automatically.
- **`late` keyword** — for deferred non-nullable initialization.
- **`required` keyword** — marks named parameters as mandatory.

---

## Q4. What are the different types of constructors in Dart?

**Answer:**

```dart
class User {
  String name;
  int age;

  // 1. Default / Generative constructor
  User(this.name, this.age);

  // 2. Named constructor
  User.guest() : name = 'Guest', age = 0;

  // 3. Redirecting constructor
  User.admin(String name) : this(name, 99);

  // 4. Constant constructor (all fields must be final)
  // const User(this.name, this.age);  // if class has only final fields

  // 5. Factory constructor
  factory User.fromJson(Map<String, dynamic> json) {
    return User(json['name'], json['age']);
  }
}
```

| Constructor Type | Purpose |
|-----------------|---------|
| Default | Standard object creation |
| Named | Alternate constructors with descriptive names |
| Redirecting | Delegates to another constructor |
| Constant | Creates compile-time constant objects |
| Factory | Can return existing instances, subtypes, or cached objects |

---

## Q5. What is the difference between `abstract class`, `mixin`, `extension`, and `interface` in Dart?

**Answer:**

```dart
// Abstract class — cannot be instantiated, can have implementation
abstract class Animal {
  void breathe() => print('breathing'); // has implementation
  void makeSound(); // abstract method — must be overridden
}

// Mixin — reusable behavior, no constructor
mixin Swimmer {
  void swim() => print('swimming');
}

// Extension — add methods to existing types without modifying them
extension StringX on String {
  String capitalize() => '${this[0].toUpperCase()}${substring(1)}';
}

// Interface — Dart has implicit interfaces; every class defines one
class Logger {
  void log(String msg) => print(msg);
}
class FileLogger implements Logger {
  @override
  void log(String msg) => print('File: $msg'); // must implement ALL methods
}
```

| Feature | Can instantiate? | Has constructor? | Multiple inheritance? |
|---------|-----------------|-----------------|----------------------|
| Abstract class | No | Yes | No (single extends) |
| Mixin | No | No | Yes (multiple with) |
| Extension | N/A | No | N/A |
| Interface (implements) | N/A | N/A | Yes (multiple implements) |

---

## Q6. Explain Dart's async programming model — Future, async/await, Stream.

**Answer:**

**Futures (single async value):**
```dart
Future<String> fetchData() async {
  final response = await http.get(Uri.parse('https://api.example.com'));
  return response.body;
}

// Error handling:
try {
  final data = await fetchData();
} catch (e) {
  print('Error: $e');
}
```

**Streams (sequence of async values):**
```dart
// Single-subscription stream
Stream<int> countStream(int max) async* {
  for (int i = 0; i < max; i++) {
    yield i;
    await Future.delayed(Duration(seconds: 1));
  }
}

// Broadcast stream — multiple listeners
final controller = StreamController<int>.broadcast();

// Listening:
countStream(5).listen(
  (data) => print(data),
  onError: (e) => print('Error: $e'),
  onDone: () => print('Done'),
);
```

**Key differences:**

| Feature | Future | Stream |
|---------|--------|--------|
| Values | Single | Multiple over time |
| Use case | API call, file read | WebSocket, sensor data, real-time updates |
| Keywords | `async`, `await` | `async*`, `yield`, `yield*` |

---

## Q7. What is an Isolate in Dart? How does it differ from threads?

**Answer:**
Dart is single-threaded. An **Isolate** is a separate memory space with its own event loop — it does NOT share memory with the main isolate.

```dart
// Using compute (simplified isolate for Flutter)
final result = await compute(expensiveFunction, data);

// Using Isolate.spawn (Dart 2.19+)
await Isolate.run(() {
  return heavyComputation();
});

// Manual Isolate with message passing
final receivePort = ReceivePort();
await Isolate.spawn((SendPort sendPort) {
  sendPort.send('Hello from isolate');
}, receivePort.sendPort);
```

| Feature | Isolate | Thread (Java/C++) |
|---------|---------|-------------------|
| Memory | Separate (no sharing) | Shared |
| Communication | Message passing (SendPort/ReceivePort) | Shared memory + locks |
| Concurrency issues | No race conditions | Requires synchronization |

**When to use Isolates:**
- JSON parsing of large data
- Image processing
- Complex mathematical computations
- Anything taking >16ms (to avoid UI jank)

---

## Q8. Explain Dart Collections — List, Set, Map and their operations.

**Answer:**

```dart
// List (ordered, duplicates allowed)
var list = [1, 2, 3, 2];
list.add(4);
list.where((e) => e > 2);       // [3, 4]
list.map((e) => e * 2);          // [2, 4, 6, 4, 8]
list.fold(0, (sum, e) => sum + e); // 12

// Set (unordered, no duplicates)
var set = {1, 2, 3};
set.add(2);  // still {1, 2, 3}
set.intersection({2, 3, 4});    // {2, 3}
set.union({4, 5});              // {1, 2, 3, 4, 5}

// Map (key-value pairs)
var map = {'name': 'Alice', 'age': 30};
map['email'] = 'alice@example.com';
map.entries;    // Iterable of MapEntry
map.keys;       // ['name', 'age', 'email']
map.values;     // ['Alice', 30, 'alice@example.com']
map.putIfAbsent('name', () => 'Bob'); // won't overwrite
```

**Spread operator and collection if/for:**
```dart
var list1 = [1, 2, 3];
var list2 = [0, ...list1, 4];  // [0, 1, 2, 3, 4]

var nav = [
  'Home',
  'Products',
  if (isAdmin) 'Admin',        // collection if
  for (var page in extraPages) page, // collection for
];
```

---

## Q9. What are Generics in Dart? Why are they important?

**Answer:**
Generics allow you to write reusable, type-safe code that works with multiple types.

```dart
// Generic class
class Box<T> {
  T value;
  Box(this.value);
}
var intBox = Box<int>(42);
var stringBox = Box<String>('hello');

// Generic method
T first<T>(List<T> items) => items[0];

// Bounded generics
class NumberBox<T extends num> {
  T value;
  NumberBox(this.value);
  double get doubled => value * 2.0;
}

// Generic typedef
typedef JsonMap = Map<String, dynamic>;
typedef Mapper<T> = T Function(Map<String, dynamic>);
```

**Benefits:** Type safety at compile time, code reuse, no casting needed.

---

## Q10. Explain the Dart Event Loop.

**Answer:**
Dart runs on a single thread with an event loop containing two queues:

1. **Microtask Queue** (higher priority) — `scheduleMicrotask()`, `Future.microtask()`
2. **Event Queue** (lower priority) — I/O, timers, `Future()`, UI events

**Execution order:**
```
1. Synchronous code runs first
2. Microtask queue is drained completely
3. One event from the event queue is processed
4. Repeat from step 2
```

```dart
print('1 - sync');
Future(() => print('5 - event queue'));
Future.microtask(() => print('3 - microtask'));
scheduleMicrotask(() => print('4 - microtask'));
print('2 - sync');

// Output: 1, 2, 3, 4, 5
```

---

## Q11. What are Records and Patterns in Dart 3?

**Answer:**

**Records** — anonymous immutable composite types:
```dart
(String, int) person = ('Alice', 30);
print(person.$1); // Alice
print(person.$2); // 30

// Named fields:
({String name, int age}) person2 = (name: 'Bob', age: 25);
print(person2.name); // Bob
```

**Patterns** — destructuring and matching:
```dart
// Destructuring
var (name, age) = ('Alice', 30);

// Switch with patterns
switch (shape) {
  case Circle(radius: var r) when r > 0:
    print('Circle with radius $r');
  case Square(side: var s):
    print('Square with side $s');
}

// If-case
if (json case {'name': String name, 'age': int age}) {
  print('$name is $age');
}
```

---

## Q12. What are Sealed Classes in Dart 3?

**Answer:**
Sealed classes restrict which classes can extend or implement them — all subtypes must be in the same library. This enables **exhaustive pattern matching**.

```dart
sealed class Shape {}
class Circle extends Shape { final double radius; Circle(this.radius); }
class Square extends Shape { final double side; Square(this.side); }
class Triangle extends Shape { final double base, height; Triangle(this.base, this.height); }

// Exhaustive switch — compiler ensures all subtypes are handled
double area(Shape shape) => switch (shape) {
  Circle(radius: var r) => 3.14 * r * r,
  Square(side: var s) => s * s,
  Triangle(base: var b, height: var h) => 0.5 * b * h,
};
// No default needed — compiler knows all cases are covered
```

---

## Q13. What is the difference between `==` and `identical()` in Dart?

**Answer:**
- `==` checks **value equality** (can be overridden).
- `identical()` checks **reference equality** (same object in memory).

```dart
var a = [1, 2, 3];
var b = [1, 2, 3];
var c = a;

print(a == b);          // true (same values via ListEquality)
print(identical(a, b)); // false (different objects)
print(identical(a, c)); // true (same reference)

// For const objects:
const x = Point(1, 2);
const y = Point(1, 2);
print(identical(x, y)); // true (compile-time constants are canonicalized)
```

---

## Q14. Explain `typedef` and `Function` types in Dart.

**Answer:**

```dart
// Function type alias
typedef IntCallback = void Function(int value);
typedef JsonParser<T> = T Function(Map<String, dynamic> json);

// Usage
void processNumbers(List<int> numbers, IntCallback callback) {
  for (var n in numbers) {
    callback(n);
  }
}
processNumbers([1, 2, 3], (value) => print(value * 2));

// Inline function types work too
void doSomething(void Function(String) onComplete) {
  onComplete('done');
}
```

---

## Q15. What are Extension Methods and Extension Types in Dart?

**Answer:**

**Extension methods** — add functionality to existing types:
```dart
extension DateTimeX on DateTime {
  String get formatted => '$day/$month/$year';
  bool get isWeekend => weekday == 6 || weekday == 7;
}
print(DateTime.now().formatted);   // 6/3/2026
print(DateTime.now().isWeekend);   // true/false
```

**Extension types** (Dart 3.3+) — zero-cost wrappers with a different API:
```dart
extension type UserId(int id) {
  bool get isValid => id > 0;
}
UserId userId = UserId(42);
// userId is still an int at runtime (zero overhead)
// but at compile time, you can only use UserId's API
```

---

## Q16. How does Dart handle error/exception handling?

**Answer:**

```dart
// Try-catch-finally
try {
  var result = riskyOperation();
} on FormatException catch (e) {
  // Specific exception type
  print('Format error: $e');
} on HttpException catch (e, stackTrace) {
  // With stack trace
  print('HTTP error: $e');
  print(stackTrace);
} catch (e) {
  // Any other exception
  rethrow; // re-throw the original exception
} finally {
  // Always executes
  cleanup();
}

// Custom exceptions
class AppException implements Exception {
  final String message;
  final int code;
  AppException(this.message, this.code);
  @override
  String toString() => 'AppException($code): $message';
}
throw AppException('Not found', 404);
```

**Note:** Dart has `Exception` (recoverable) and `Error` (programming bugs like `RangeError`). Catch `Exception`s, don't catch `Error`s.

---

## Q17. What is the `cascade` operator (`..`) in Dart?

**Answer:**
The cascade operator allows you to chain multiple operations on the same object without repeating the variable name.

```dart
// Without cascade
var paint = Paint();
paint.color = Colors.red;
paint.strokeWidth = 2.0;
paint.style = PaintingStyle.stroke;

// With cascade
var paint = Paint()
  ..color = Colors.red
  ..strokeWidth = 2.0
  ..style = PaintingStyle.stroke;

// Null-aware cascade
canvas?..drawLine(a, b, paint)
       ..drawCircle(center, radius, paint);
```

---

## Q18. Explain `Iterable` vs `List` in Dart.

**Answer:**

| Feature | Iterable | List |
|---------|----------|------|
| Lazy evaluation | Yes | No |
| Index access | No (`elementAt()` is O(n)) | Yes (`list[i]` is O(1)) |
| Methods like `where()`, `map()` | Return `Iterable` (lazy) | Return `Iterable` (lazy) |
| Use case | Processing pipelines | Direct access by index |

```dart
var numbers = [1, 2, 3, 4, 5];
// Lazy — doesn't execute until iterated
Iterable<int> evens = numbers.where((n) => n.isEven);
// Force evaluation to List
List<int> evenList = evens.toList();
```

---

## Q19. What are `Enum` enhancements in Dart?

**Answer:**

```dart
enum Status {
  active('Active', Colors.green),
  inactive('Inactive', Colors.grey),
  banned('Banned', Colors.red);

  final String label;
  final Color color;

  const Status(this.label, this.color);

  // Methods on enums
  bool get isUsable => this != Status.banned;

  // Implementing interfaces
  // enum Status implements Comparable<Status> { ... }
}

print(Status.active.label);    // Active
print(Status.active.isUsable); // true
print(Status.values);          // [active, inactive, banned]
print(Status.active.name);     // active (string name)
print(Status.active.index);    // 0
```

---

## Q20. What are `class modifiers` in Dart 3?

**Answer:**

| Modifier | Extend? | Implement? | Mixin? | Construct? |
|----------|---------|------------|--------|------------|
| `class` | ✅ | ✅ | ❌ | ✅ |
| `abstract class` | ✅ | ✅ | ❌ | ❌ |
| `base class` | ✅ (same library or subclasses) | ❌ | ❌ | ✅ |
| `interface class` | ❌ | ✅ | ❌ | ✅ |
| `final class` | ❌ | ❌ | ❌ | ✅ |
| `sealed class` | ✅ (same library only) | ✅ (same library only) | ❌ | ❌ |
| `mixin class` | ✅ | ✅ | ✅ | ✅ |

```dart
base class Animal {}           // can only be extended, not implemented
interface class Printable {}   // can only be implemented, not extended
final class Config {}          // cannot be extended or implemented outside library
sealed class Result {}         // only subtyped in same file, enables exhaustive switches
mixin class Walker {}          // can be used as both mixin and class
```
