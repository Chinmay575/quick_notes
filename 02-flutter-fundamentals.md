# 02 — Flutter Fundamentals

---

## Q1. What is Flutter? How does it work under the hood?

**Answer:**
Flutter is Google's open-source UI toolkit for building natively compiled applications for mobile, web, and desktop from a single codebase.

**Architecture layers:**
1. **Framework (Dart)** — Widgets, Material/Cupertino, rendering, animation, gestures
2. **Engine (C++)** — Skia/Impeller (rendering), Dart runtime, text layout, platform channels
3. **Embedder (Platform-specific)** — Integrates with OS (Android Activity, iOS UIViewController)

**How it renders:**
- Flutter does NOT use OEM widgets (UIKit/Android Views).
- It paints every pixel using its own rendering engine (Skia or Impeller).
- Widget → Element → RenderObject pipeline.
- Runs at 60/120 FPS with its own compositor.

---

## Q2. Explain the Widget Tree, Element Tree, and RenderObject Tree.

**Answer:**
Flutter has **three trees**:

| Tree | Purpose | Mutable? | Lifecycle |
|------|---------|----------|-----------|
| **Widget Tree** | Configuration/blueprint (what to display) | Immutable | Rebuilt frequently |
| **Element Tree** | Links widgets to render objects (the glue) | Mutable | Long-lived, updated in-place |
| **RenderObject Tree** | Layout, painting, hit testing (actual rendering) | Mutable | Long-lived |

```
Widget (const Text('Hello'))
  ↓ createElement()
Element (TextElement)
  ↓ createRenderObject()
RenderObject (RenderParagraph) → handles layout & paint
```

**Key insight:** When `setState()` is called, widgets are rebuilt (cheap), but Elements and RenderObjects are reused when possible (via `canUpdate()` which checks `runtimeType` and `key`).

---

## Q3. What is the difference between StatelessWidget and StatefulWidget?

**Answer:**

| Feature | StatelessWidget | StatefulWidget |
|---------|----------------|----------------|
| State | No mutable state | Has mutable State object |
| Rebuild | Only when parent rebuilds | `setState()`, or parent rebuild |
| Lifecycle | `build()` only | `initState()`, `build()`, `dispose()`, etc. |
| Use case | Static UI, displays data | Interactive UI, animations, forms |

```dart
// StatelessWidget
class Greeting extends StatelessWidget {
  final String name;
  const Greeting({required this.name});

  @override
  Widget build(BuildContext context) => Text('Hello $name');
}

// StatefulWidget
class Counter extends StatefulWidget {
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => setState(() => _count++),
      child: Text('Count: $_count'),
    );
  }
}
```

---

## Q4. Explain the StatefulWidget lifecycle methods.

**Answer:**

```dart
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // Called ONCE when State is created
    // Initialize controllers, subscriptions, fetch data
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // Called after initState and when InheritedWidget changes
    // Safe to call Theme.of(context), MediaQuery.of(context) here
  }

  @override
  void didUpdateWidget(covariant MyWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Called when parent rebuilds with new widget (same runtimeType + key)
    // Compare old/new widget properties, update accordingly
  }

  @override
  Widget build(BuildContext context) {
    // Called frequently — must be PURE (no side effects)
    return Container();
  }

  @override
  void deactivate() {
    super.deactivate();
    // Called when State is removed from tree (may be reinserted)
  }

  @override
  void dispose() {
    // Called when State is permanently removed
    // Cancel subscriptions, dispose controllers, close streams
    _controller.dispose();
    _subscription.cancel();
    super.dispose();
  }
}
```

**Order:** `createState()` → `initState()` → `didChangeDependencies()` → `build()` → ... → `deactivate()` → `dispose()`

---

## Q5. What is BuildContext? Why is it important?

**Answer:**
`BuildContext` is a reference to the location of a widget in the Element tree. Every widget has a context.

**What it provides:**
- Access to the widget's position in the tree
- Lookup ancestor widgets: `Theme.of(context)`, `MediaQuery.of(context)`
- Navigation: `Navigator.of(context)`
- Scaffold access: `ScaffoldMessenger.of(context)`

**Common mistakes:**
```dart
@override
void initState() {
  super.initState();
  // ❌ WRONG — context is not fully ready in initState
  // Theme.of(context); // may not work as expected

  // ✅ Use didChangeDependencies or addPostFrameCallback
  WidgetsBinding.instance.addPostFrameCallback((_) {
    Theme.of(context); // safe here
  });
}
```

**Key rule:** `context` represents the Element for THIS widget. `Theme.of(context)` looks UP the tree from this point.

---

## Q6. What are Keys in Flutter? When should you use them?

**Answer:**
Keys preserve state when widgets move in the tree. Without keys, Flutter matches by **type + position**.

**Types of keys:**

| Key Type | Use Case |
|----------|----------|
| `ValueKey` | Based on a unique value (ID, name) |
| `ObjectKey` | Based on object identity |
| `UniqueKey` | Forces new state every time |
| `GlobalKey` | Access state/context across the tree |
| `PageStorageKey` | Preserve scroll position |

```dart
// ❌ Without key — state gets confused when reordering
ListView(children: items.map((item) => ListTile(title: Text(item))).toList());

// ✅ With key — state follows the correct widget
ListView(children: items.map((item) =>
  ListTile(key: ValueKey(item.id), title: Text(item.name))
).toList());

// GlobalKey — access child state from parent
final _formKey = GlobalKey<FormState>();
Form(key: _formKey, child: ...);
_formKey.currentState?.validate();
```

**When to use keys:**
- Reordering lists (always use keys in list items)
- Adding/removing items from stateful lists
- Swapping widgets of the same type
- Preserving state across tree restructuring

---

## Q7. What is the difference between `const` and `final` widgets?

**Answer:**

```dart
// const widget — compile-time constant, never rebuilt
const Text('Hello'); // same instance reused everywhere

// Regular widget — rebuilt each time parent rebuilds
Text('Hello');
```

**Benefits of const constructors:**
- Widget is created ONCE and reused → less garbage collection
- Flutter skips rebuilding entire const subtrees
- Can improve performance significantly in large lists

```dart
class MyWidget extends StatelessWidget {
  const MyWidget({super.key}); // const constructor

  @override
  Widget build(BuildContext context) {
    return const Column(
      children: [
        Text('Static text'),      // const — never rebuilt
        Icon(Icons.star),         // const — never rebuilt
      ],
    );
  }
}
```

**Rule of thumb:** Use `const` everywhere possible. Enable the `prefer_const_constructors` lint.

---

## Q8. Explain InheritedWidget. How does it work?

**Answer:**
`InheritedWidget` is the mechanism for passing data down the widget tree efficiently. All `.of(context)` patterns use it (Theme, MediaQuery, Navigator).

```dart
class UserData extends InheritedWidget {
  final String username;
  final String email;

  const UserData({
    required this.username,
    required this.email,
    required super.child,
  });

  // Convenience method
  static UserData of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<UserData>()!;
  }

  @override
  bool updateShouldNotify(UserData oldWidget) {
    return username != oldWidget.username || email != oldWidget.email;
  }
}

// Usage
UserData(
  username: 'Alice',
  email: 'alice@example.com',
  child: MyApp(),
);

// Access from any descendant
final user = UserData.of(context);
print(user.username);
```

**How it works:**
1. `dependOnInheritedWidgetOfExactType()` registers the calling widget as a dependent.
2. When `updateShouldNotify()` returns true, all dependents are rebuilt.
3. O(1) lookup — stored in a HashMap on the Element.

---

## Q9. What are Slivers in Flutter?

**Answer:**
Slivers are scrollable areas that can lazily build their content. They're the building blocks of `CustomScrollView`.

```dart
CustomScrollView(
  slivers: [
    // App bar that collapses on scroll
    SliverAppBar(
      expandedHeight: 200,
      floating: true,
      flexibleSpace: FlexibleSpaceBar(title: Text('Title')),
    ),

    // Grid
    SliverGrid(
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 2),
      delegate: SliverChildBuilderDelegate(
        (context, index) => Card(child: Text('Item $index')),
        childCount: 20,
      ),
    ),

    // List
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, index) => ListTile(title: Text('Item $index')),
        childCount: 100,
      ),
    ),

    // Fixed-height items (more efficient)
    SliverFixedExtentList(
      itemExtent: 60.0,
      delegate: SliverChildBuilderDelegate(
        (context, index) => ListTile(title: Text('Item $index')),
      ),
    ),
  ],
)
```

**Common slivers:** `SliverAppBar`, `SliverList`, `SliverGrid`, `SliverToBoxAdapter`, `SliverFillRemaining`, `SliverPersistentHeader`, `SliverAnimatedList`.

---

## Q10. What is the difference between `mainAxisAlignment` and `crossAxisAlignment`?

**Answer:**

| Property | Row | Column |
|----------|-----|--------|
| `mainAxisAlignment` | Horizontal ↔ | Vertical ↕ |
| `crossAxisAlignment` | Vertical ↕ | Horizontal ↔ |

**MainAxisAlignment values:**
- `start` — pack children at the start
- `end` — pack at the end
- `center` — center children
- `spaceBetween` — equal space between children
- `spaceAround` — equal space around each child
- `spaceEvenly` — equal space between and at edges

**CrossAxisAlignment values:**
- `start`, `end`, `center`, `stretch`, `baseline`

---

## Q11. Explain `Expanded`, `Flexible`, and `Spacer` widgets.

**Answer:**

```dart
Row(
  children: [
    // Expanded — takes all remaining space (flex factor)
    Expanded(
      flex: 2, // takes 2/3 of remaining space
      child: Container(color: Colors.red),
    ),

    // Flexible — takes up to remaining space, but can be smaller
    Flexible(
      flex: 1, // takes up to 1/3 of remaining space
      fit: FlexFit.loose, // can be smaller than allocated space
      child: Container(color: Colors.blue),
    ),

    // Spacer — empty expanded widget
    Spacer(flex: 1), // equivalent to Expanded(child: SizedBox.shrink())
  ],
)
```

| Widget | Fills space? | Shrinks? |
|--------|-------------|----------|
| `Expanded` | Yes (always fills allocated space) | No |
| `Flexible(fit: FlexFit.loose)` | No (can be smaller) | Yes |
| `Flexible(fit: FlexFit.tight)` | Yes (same as Expanded) | No |
| `Spacer` | Yes (empty space) | No |

---

## Q12. What is `MediaQuery` and how is it used for responsive design?

**Answer:**

```dart
// Get screen info
final size = MediaQuery.of(context).size;
final width = size.width;
final height = size.height;
final orientation = MediaQuery.of(context).orientation;
final padding = MediaQuery.of(context).padding; // safe area
final textScale = MediaQuery.of(context).textScaleFactor;
final brightness = MediaQuery.of(context).platformBrightness;

// Responsive layout
Widget build(BuildContext context) {
  final width = MediaQuery.of(context).size.width;

  if (width > 1200) return DesktopLayout();
  if (width > 600) return TabletLayout();
  return MobileLayout();
}

// LayoutBuilder — responds to parent constraints
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) {
      return WideLayout();
    }
    return NarrowLayout();
  },
)
```

**MediaQuery vs LayoutBuilder:**
- `MediaQuery` — gives screen-level info (entire screen size, padding, etc.)
- `LayoutBuilder` — gives the constraints from the parent widget (more granular)

---

## Q13. What are `GlobalKey`, `LocalKey`, and `PageStorageKey`?

**Answer:**

**LocalKey** (positioned within parent):
```dart
// ValueKey — identity based on value
ListTile(key: ValueKey(user.id), title: Text(user.name));

// ObjectKey — identity based on object reference
ListTile(key: ObjectKey(user), title: Text(user.name));

// UniqueKey — always unique, forces rebuild
Container(key: UniqueKey());
```

**GlobalKey** (unique across entire app):
```dart
// Access state from anywhere
final scaffoldKey = GlobalKey<ScaffoldState>();
Scaffold(key: scaffoldKey, ...);
scaffoldKey.currentState?.openDrawer();

// Access size/position
final key = GlobalKey();
Container(key: key);
final renderBox = key.currentContext?.findRenderObject() as RenderBox;
final position = renderBox.localToGlobal(Offset.zero);
```

**PageStorageKey** (preserves scroll position):
```dart
ListView(
  key: PageStorageKey('my-list'),
  children: [...],
); // Scroll position survives tab switches
```

---

## Q14. What is the difference between `hot reload` and `hot restart`?

**Answer:**

| Feature | Hot Reload | Hot Restart |
|---------|-----------|-------------|
| Speed | ~1 second | ~3-5 seconds |
| State preserved | ✅ Yes | ❌ No (resets to initial) |
| What changes | `build()` methods, UI code | Everything including `main()`, `initState()` |
| When to use | UI tweaks, minor logic changes | Static field changes, enum changes, generic type changes |
| Limitations | Can't change `initState`, `main()`, static fields | Full app restart without reinstalling |

**Hot reload won't work when:**
- Changing `main()` function
- Changing static fields or initializers
- Changing enum values
- Changing generic type arguments
- Changing native code (platform channels)

---

## Q15. What are `WidgetsBinding` and `WidgetsBindingObserver`?

**Answer:**

```dart
class _MyAppState extends State<MyApp> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        // App is in foreground
        break;
      case AppLifecycleState.inactive:
        // App is transitioning (e.g., phone call overlay)
        break;
      case AppLifecycleState.paused:
        // App is in background
        break;
      case AppLifecycleState.detached:
        // App is being terminated
        break;
      case AppLifecycleState.hidden:
        // App is hidden (Dart 3.13+)
        break;
    }
  }

  @override
  void didChangeMetrics() {
    // Screen size, orientation, or other metrics changed
  }

  @override
  void didChangeLocales(List<Locale>? locales) {
    // Device locale changed
  }

  @override
  void didChangeTextScaleFactor() {
    // Text scale factor changed (accessibility)
  }
}
```

---

## Q16. What is `RepaintBoundary` and when should you use it?

**Answer:**
`RepaintBoundary` creates a separate painting layer, preventing repaints from propagating to parent or sibling widgets.

```dart
// Without RepaintBoundary — animation causes entire parent to repaint
Stack(
  children: [
    ExpensiveBackground(),
    AnimatedWidget(), // every frame repaints ExpensiveBackground too!
  ],
)

// With RepaintBoundary — only AnimatedWidget repaints
Stack(
  children: [
    RepaintBoundary(child: ExpensiveBackground()), // isolated layer
    AnimatedWidget(),
  ],
)
```

**When to use:**
- Animated widgets that repaint frequently
- Complex static widgets that shouldn't be repainted
- Scrolling lists (ListView already adds them internally)
- Use `debugRepaintRainbowEnabled = true` to visualize repaints

---

## Q17. Explain the constraint system in Flutter (BoxConstraints).

**Answer:**
Flutter layout uses a **constraint-based** system:

1. **Constraints go DOWN** — parent tells child its min/max width/height
2. **Sizes go UP** — child picks a size within constraints
3. **Parent sets position** — parent decides where to place child

```dart
// Tight constraints — forces exact size
BoxConstraints.tight(Size(100, 100)) // min=max=100

// Loose constraints — allows any size up to max
BoxConstraints.loose(Size(100, 100)) // min=0, max=100

// Common issue: unbounded constraints
// ❌ Column inside Column without constraints
Column(children: [
  Column(children: [ /* RenderFlex overflow error */ ]),
]);

// ✅ Wrap in Expanded or give constraints
Column(children: [
  Expanded(child: Column(children: [...])),
]);
```

**"Constraints go down, sizes go up, parent sets position"** — this is Flutter's golden rule of layout.

---

## Q18. What is `FutureBuilder` and `StreamBuilder`?

**Answer:**

```dart
// FutureBuilder — builds UI based on Future result
FutureBuilder<User>(
  future: fetchUser(), // should NOT be called in build()
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return CircularProgressIndicator();
    }
    if (snapshot.hasError) {
      return Text('Error: ${snapshot.error}');
    }
    if (snapshot.hasData) {
      return Text('Hello ${snapshot.data!.name}');
    }
    return SizedBox.shrink();
  },
)

// StreamBuilder — builds UI based on Stream data
StreamBuilder<int>(
  stream: counterStream(),
  initialData: 0,
  builder: (context, snapshot) {
    return Text('Count: ${snapshot.data}');
  },
)
```

**⚠️ Important:** Store the Future in `initState()` or a variable — don't call it directly in `build()`, or it'll create a new Future on every rebuild.

```dart
late final Future<User> _userFuture;

@override
void initState() {
  super.initState();
  _userFuture = fetchUser(); // called once
}

@override
Widget build(BuildContext context) {
  return FutureBuilder(future: _userFuture, ...); // reuses same future
}
```

---

## Q19. What is `SafeArea` and why is it important?

**Answer:**
`SafeArea` insets its child to avoid system UI intrusions (notch, status bar, navigation bar, camera cutout).

```dart
Scaffold(
  body: SafeArea(
    // Customizable sides
    top: true,      // avoid status bar
    bottom: true,   // avoid home indicator
    left: true,     // avoid notch (landscape)
    right: true,
    minimum: EdgeInsets.all(16), // minimum padding
    child: MyContent(),
  ),
)
```

Under the hood, `SafeArea` uses `MediaQuery.of(context).padding` to determine safe insets.

---

## Q20. What is the difference between `Navigator.push` and `Navigator.pushReplacement`?

**Answer:**

| Method | Stack behavior | Back button |
|--------|---------------|-------------|
| `push` | Adds on top of stack | Returns to previous screen |
| `pushReplacement` | Replaces current screen | Cannot go back to replaced screen |
| `pushAndRemoveUntil` | Pushes and removes screens below | Goes back to specified condition |
| `pop` | Removes top screen | N/A |
| `popUntil` | Pops until condition is met | N/A |
| `pushNamed` | Push using route name | Returns to previous screen |

```dart
// Push new screen
Navigator.push(context, MaterialPageRoute(builder: (_) => DetailScreen()));

// Replace current screen (e.g., login → home)
Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => HomeScreen()));

// Clear stack and push (e.g., logout)
Navigator.pushAndRemoveUntil(
  context,
  MaterialPageRoute(builder: (_) => LoginScreen()),
  (route) => false, // removes ALL routes
);

// Return data
final result = await Navigator.push<String>(
  context,
  MaterialPageRoute(builder: (_) => SelectionScreen()),
);
Navigator.pop(context, 'selected_value'); // from SelectionScreen
```
