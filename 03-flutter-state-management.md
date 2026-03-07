# 03 — Flutter State Management

---

## Q1. What is state management in Flutter? Why do we need it?

**Answer:**
State management is the strategy for managing, sharing, and updating data that affects the UI. Flutter rebuilds widgets when state changes — managing WHERE state lives and HOW it propagates is critical.

**Two types of state:**
- **Ephemeral (local) state** — affects a single widget (tab index, animation progress). Use `setState()`.
- **App state** — shared across widgets/screens (user auth, cart, settings). Use a state management solution.

**Why we need it:**
- Passing data through many widget layers is tedious ("prop drilling")
- Multiple widgets need to react to the same data changes
- Business logic should be separated from UI
- State needs to survive widget rebuilds

---

## Q2. Explain `setState()` — when to use and when NOT to use it.

**Answer:**

```dart
class _CounterState extends State<Counter> {
  int _count = 0;

  void _increment() {
    setState(() {
      _count++;
    });
    // setState marks this widget as dirty → triggers build()
  }

  @override
  Widget build(BuildContext context) => Text('$_count');
}
```

**When to use:**
- Simple, local state within a single widget
- Toggle switches, form validation, tab selection
- Animation state

**When NOT to use:**
- State shared across multiple widgets
- Complex state with many interdependent values
- State that needs to survive screen navigation
- When you need to separate business logic from UI

**Common mistakes:**
```dart
// ❌ async gap — widget might be disposed
void _load() async {
  final data = await fetchData();
  setState(() { _data = data; }); // may crash if widget disposed
}

// ✅ Check mounted
void _load() async {
  final data = await fetchData();
  if (mounted) setState(() { _data = data; });
}
```

---

## Q3. Explain Provider state management.

**Answer:**
Provider is an InheritedWidget wrapper that simplifies dependency injection and state management.

```dart
// 1. Define a ChangeNotifier
class CartModel extends ChangeNotifier {
  final List<Item> _items = [];
  List<Item> get items => UnmodifiableListView(_items);

  void add(Item item) {
    _items.add(item);
    notifyListeners(); // notifies all listeners to rebuild
  }

  void removeAll() {
    _items.clear();
    notifyListeners();
  }
}

// 2. Provide it
void main() {
  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => CartModel()),
        ChangeNotifierProvider(create: (_) => UserModel()),
        Provider(create: (_) => ApiService()),
      ],
      child: MyApp(),
    ),
  );
}

// 3. Consume it
// Read (no rebuild on changes)
final cart = context.read<CartModel>();
cart.add(item);

// Watch (rebuilds when CartModel changes)
final items = context.watch<CartModel>().items;

// Select (rebuilds only when selected value changes)
final itemCount = context.select<CartModel, int>((cart) => cart.items.length);

// Consumer widget (scoped rebuilds)
Consumer<CartModel>(
  builder: (context, cart, child) {
    return Text('${cart.items.length} items');
  },
)
```

**Provider types:**

| Provider | Use Case |
|----------|----------|
| `Provider` | Immutable value / service |
| `ChangeNotifierProvider` | Mutable state with `notifyListeners()` |
| `FutureProvider` | Async value from a Future |
| `StreamProvider` | Async value from a Stream |
| `ProxyProvider` | Depends on another provider |

---

## Q4. Explain Riverpod state management.

**Answer:**
Riverpod is a reactive caching and data-binding framework — a complete rewrite of Provider without BuildContext dependency.

```dart
// Define providers (global, but safe)
final counterProvider = StateNotifierProvider<CounterNotifier, int>((ref) {
  return CounterNotifier();
});

class CounterNotifier extends StateNotifier<int> {
  CounterNotifier() : super(0);
  void increment() => state++;
  void decrement() => state--;
}

// Async data
final userProvider = FutureProvider<User>((ref) async {
  final repo = ref.watch(repositoryProvider);
  return repo.fetchUser();
});

// Using in widget
class CounterPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Column(
      children: [
        Text('Count: $count'),
        ElevatedButton(
          onPressed: () => ref.read(counterProvider.notifier).increment(),
          child: Text('Increment'),
        ),
      ],
    );
  }
}

// AsyncValue handling
final userData = ref.watch(userProvider);
return userData.when(
  data: (user) => Text(user.name),
  loading: () => CircularProgressIndicator(),
  error: (err, stack) => Text('Error: $err'),
);
```

**Riverpod Provider types:**
- `Provider` — read-only value
- `StateProvider` — simple mutable state
- `StateNotifierProvider` — complex state with methods
- `FutureProvider` — async value
- `StreamProvider` — stream value
- `NotifierProvider` (Riverpod 2.0) — modern replacement for StateNotifier

**Advantages over Provider:**
- No BuildContext needed
- Compile-time safety
- Multiple providers of the same type
- Provider auto-dispose
- Better testing

---

## Q5. Explain BLoC (Business Logic Component) pattern.

**Answer:**
BLoC separates business logic from UI using Streams (Events in → States out).

```dart
// Events
abstract class CounterEvent {}
class IncrementEvent extends CounterEvent {}
class DecrementEvent extends CounterEvent {}

// States
class CounterState {
  final int count;
  CounterState(this.count);
}

// BLoC
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(CounterState(0)) {
    on<IncrementEvent>((event, emit) {
      emit(CounterState(state.count + 1));
    });

    on<DecrementEvent>((event, emit) {
      emit(CounterState(state.count - 1));
    });
  }
}

// Provide
BlocProvider(
  create: (_) => CounterBloc(),
  child: CounterPage(),
)

// Consume
BlocBuilder<CounterBloc, CounterState>(
  builder: (context, state) {
    return Text('Count: ${state.count}');
  },
)

// Listen for side effects (navigation, snackbar)
BlocListener<CounterBloc, CounterState>(
  listener: (context, state) {
    if (state.count == 10) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Reached 10!')),
      );
    }
  },
  child: ...,
)

// Dispatch event
context.read<CounterBloc>().add(IncrementEvent());
```

---

## Q6. What is a Cubit? How does it differ from BLoC?

**Answer:**

```dart
// Cubit — simplified BLoC without events
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
}

// Usage is identical to BLoC
BlocProvider(create: (_) => CounterCubit(), child: MyWidget())

BlocBuilder<CounterCubit, int>(
  builder: (context, count) => Text('$count'),
)

context.read<CounterCubit>().increment();
```

| Feature | Cubit | BLoC |
|---------|-------|------|
| Input | Method calls | Events |
| Traceability | Less (direct calls) | More (event log) |
| Boilerplate | Less | More (events + handler) |
| Testability | Simpler | Better (event-driven) |
| Use case | Simple state | Complex state with event transformations |

---

## Q7. Explain GetX state management.

**Answer:**

```dart
// Controller
class CounterController extends GetxController {
  var count = 0.obs; // .obs makes it observable (Rx)

  void increment() => count++;

  // Lifecycle
  @override
  void onInit() { super.onInit(); /* init logic */ }
  @override
  void onClose() { super.onClose(); /* cleanup */ }
}

// Simple State Manager
class SimpleController extends GetxController {
  int count = 0;
  void increment() {
    count++;
    update(); // triggers rebuild for GetBuilder
  }
}

// Reactive approach
GetX<CounterController>(
  builder: (controller) => Text('${controller.count}'),
)

// Simple approach
GetBuilder<SimpleController>(
  builder: (controller) => Text('${controller.count}'),
)

// Obx — simplest reactive
Obx(() => Text('${controller.count}'))

// Dependency injection
Get.put(CounterController());           // eager
Get.lazyPut(() => CounterController());  // lazy
Get.find<CounterController>();           // retrieve
```

**Pros:** Minimal boilerplate, batteries-included (navigation, DI, HTTP).
**Cons:** Tight coupling to GetX, less testable, magic/implicit behavior.

---

## Q8. Explain Redux in Flutter.

**Answer:**

```dart
// State
class AppState {
  final int counter;
  AppState({this.counter = 0});
}

// Actions
class IncrementAction {}
class DecrementAction {}

// Reducer (pure function)
AppState appReducer(AppState state, dynamic action) {
  if (action is IncrementAction) return AppState(counter: state.counter + 1);
  if (action is DecrementAction) return AppState(counter: state.counter - 1);
  return state;
}

// Store
final store = Store<AppState>(appReducer, initialState: AppState());

// Provide
StoreProvider<AppState>(store: store, child: MyApp())

// Connect
StoreConnector<AppState, int>(
  converter: (store) => store.state.counter,
  builder: (context, count) => Text('$count'),
)

// Dispatch
StoreProvider.of<AppState>(context).dispatch(IncrementAction());
```

**Redux principles:**
1. **Single source of truth** — one store
2. **State is read-only** — only changed via actions
3. **Changes via pure functions** — reducers

---

## Q9. Compare all major state management solutions.

**Answer:**

| Solution | Complexity | Boilerplate | Learning Curve | Best For |
|----------|-----------|-------------|---------------|----------|
| `setState` | Low | None | Easy | Local widget state |
| `InheritedWidget` | Medium | High | Medium | Custom solutions |
| `Provider` | Low-Medium | Low | Easy | Most apps, recommended by Flutter |
| `Riverpod` | Medium | Medium | Medium | Type-safe, testable apps |
| `BLoC/Cubit` | Medium-High | High (BLoC), Medium (Cubit) | Medium-Hard | Large enterprise apps, team projects |
| `GetX` | Low | Very Low | Easy | Rapid prototyping |
| `Redux` | High | Very High | Hard | Apps needing time-travel debugging |
| `MobX` | Medium | Low (with codegen) | Medium | Reactive programming fans |
| `Signals` | Low | Low | Easy | Fine-grained reactivity |

---

## Q10. What is `ValueNotifier` and `ValueListenableBuilder`?

**Answer:**
`ValueNotifier` is a simpler alternative to `ChangeNotifier` for single-value state.

```dart
// Define
final counter = ValueNotifier<int>(0);

// Update
counter.value++;

// Listen in widget
ValueListenableBuilder<int>(
  valueListenable: counter,
  builder: (context, value, child) {
    return Text('Count: $value');
  },
  child: Icon(Icons.star), // child is not rebuilt
)

// Dispose
@override
void dispose() {
  counter.dispose();
  super.dispose();
}
```

**Use case:** Simple reactive values without the overhead of full state management.

---

## Q11. How do you handle state restoration in Flutter?

**Answer:**

```dart
class _MyPageState extends State<MyPage> with RestorationMixin {
  // Restorable properties
  final RestorableInt _counter = RestorableInt(0);
  final RestorableString _name = RestorableString('');
  final RestorableBool _isChecked = RestorableBool(false);

  @override
  String get restorationId => 'my_page';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_counter, 'counter');
    registerForRestoration(_name, 'name');
    registerForRestoration(_isChecked, 'is_checked');
  }

  @override
  void dispose() {
    _counter.dispose();
    _name.dispose();
    _isChecked.dispose();
    super.dispose();
  }
}
```

State restoration preserves app state when the OS kills the app in the background (common on Android).

---

## Q12. What is `ChangeNotifier` vs `StateNotifier`?

**Answer:**

| Feature | ChangeNotifier | StateNotifier |
|---------|---------------|---------------|
| State | Mutable (modify in place) | Immutable (replace state) |
| Notification | `notifyListeners()` manual call | Automatic on `state = newState` |
| Package | Flutter SDK | `state_notifier` / Riverpod |
| Testing | Harder (mutable state) | Easier (immutable snapshots) |

```dart
// ChangeNotifier — mutable
class CartModel extends ChangeNotifier {
  final List<Item> items = [];
  void add(Item item) {
    items.add(item);      // mutate in place
    notifyListeners();    // manual notification
  }
}

// StateNotifier — immutable
class CartNotifier extends StateNotifier<List<Item>> {
  CartNotifier() : super([]);
  void add(Item item) {
    state = [...state, item]; // replace entire state
    // notification is automatic
  }
}
```
