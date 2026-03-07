# 10 — Flutter Performance Optimization

---

## Q1. What causes jank in Flutter and how do you fix it?

**Answer:**
Jank = frames taking >16ms (60fps) or >8ms (120fps) to render.

**Common causes and fixes:**

| Cause | Fix |
|-------|-----|
| Expensive `build()` methods | Break into smaller widgets, use `const` |
| Rebuilding too many widgets | Use `Consumer`, `Selector`, `BlocBuilder` with `buildWhen` |
| Heavy computation on UI thread | Use `Isolate.run()` or `compute()` |
| Large images | Cache with `CachedNetworkImage`, resize, use proper format |
| Overdraw (painting hidden layers) | Use `RepaintBoundary`, minimize `Opacity` widget |
| Too many layers | Reduce `ClipRRect`, shadows, `BackdropFilter` |
| Unnecessary `setState()` | Only rebuild what changed, use granular state management |

---

## Q2. How do you optimize ListView for large lists?

**Answer:**

```dart
// ✅ ListView.builder — only builds visible items
ListView.builder(
  itemCount: 10000,
  itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
)

// ✅ Use const widgets for static content
itemBuilder: (context, index) => const Divider(),

// ✅ Use itemExtent for fixed-height items (skip layout calculation)
ListView.builder(
  itemExtent: 72.0, // all items are exactly 72px tall
  itemCount: 10000,
  itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
)

// ✅ Avoid widgets that force full-list rendering
// ❌ ListView(children: [...]) — builds ALL children
// ❌ Column + SingleChildScrollView — builds ALL children
// ✅ ListView.builder — lazy, only visible items

// ✅ Cache scroll position
ListView.builder(
  key: PageStorageKey('my-list'),
  ...
)

// ✅ Use addAutomaticKeepAlives: false for memory
ListView.builder(
  addAutomaticKeepAlives: false,
  addRepaintBoundaries: true,
  ...
)
```

---

## Q3. How do you optimize images in Flutter?

**Answer:**

```dart
// 1. Use CachedNetworkImage
CachedNetworkImage(
  imageUrl: url,
  placeholder: (_, __) => CircularProgressIndicator(),
  errorWidget: (_, __, ___) => Icon(Icons.error),
  memCacheWidth: 300, // resize in memory
)

// 2. Use proper cacheWidth/cacheHeight
Image.network(
  url,
  cacheWidth: 300,  // decode to smaller size in memory
  cacheHeight: 300,
)

// 3. Use WebP format (smaller than PNG/JPEG)
// 4. Use SVG for icons/illustrations (flutter_svg)
SvgPicture.asset('assets/icon.svg', width: 24, height: 24)

// 5. Precache critical images
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  precacheImage(AssetImage('assets/hero.png'), context);
}

// 6. Lazy load images in lists
// CachedNetworkImage does this automatically

// 7. Use AssetImage with resolution variants
// assets/images/icon.png      (1x)
// assets/images/2.0x/icon.png (2x)
// assets/images/3.0x/icon.png (3x)
```

---

## Q4. How do you reduce app size?

**Answer:**

```bash
# Build with split APKs (per ABI)
flutter build apk --split-per-abi
# Produces: app-armeabi-v7a.apk, app-arm64-v8a.apk, app-x86_64.apk

# Build App Bundle (Google Play handles splitting)
flutter build appbundle

# Analyze app size
flutter build apk --analyze-size
# Opens DevTools app size analysis

# Use --obfuscate and --split-debug-info for smaller builds
flutter build apk --obfuscate --split-debug-info=debug-info/
```

**Other techniques:**
- Remove unused packages from `pubspec.yaml`
- Use `flutter_launcher_icons` with optimized images
- Compress assets (TinyPNG for images)
- Use ProGuard/R8 (Android) — enabled by default in release
- Remove unused fonts — specify only needed font weights
- Use `--tree-shake-icons` (default in Flutter 3.x)
- Use deferred loading: `import 'package:heavy.dart' deferred as heavy;`

---

## Q5. How do you profile Flutter app performance?

**Answer:**

```bash
# Run in profile mode (release performance, debug tools)
flutter run --profile

# Enable performance overlay
showPerformanceOverlay: true, // in MaterialApp

# Check for expensive operations
Timeline.startSync('myExpensiveOperation');
doWork();
Timeline.finishSync();
```

**DevTools Performance tab:**
- **UI thread** (blue) — Dart code, build/layout
- **Raster thread** (green) — painting/compositing
- Red bars = jank (>16ms frame)

**Checklist:**
1. Profile in **release mode** (not debug — debug mode is 10x slower)
2. Profile on **real device** (not emulator)
3. Check for unnecessary rebuilds with `debugPrintRebuildDirtyWidgets = true`
4. Use `RepaintBoundary` to isolate expensive paints
5. Check memory with heap snapshots

---

## Q6. What is tree shaking and dead code elimination?

**Answer:**
Tree shaking removes unused code during compilation:
- Unused classes, functions, and imports are excluded from the final binary
- Only code reachable from `main()` is included
- Applies in **release mode** (AOT compilation)

```dart
// Icon tree shaking (Flutter 3.x default)
// Only icons actually used in code are included in the font file
// Removes unused Material/Cupertino icons → saves ~1MB

// Deferred imports for code splitting (web)
import 'package:heavy_feature/feature.dart' deferred as feature;

Future<void> loadFeature() async {
  await feature.loadLibrary();
  feature.showFeature();
}
```

---

## Q7. How do you avoid unnecessary widget rebuilds?

**Answer:**

```dart
// 1. Use const constructors
const Text('Static text'); // never rebuilt

// 2. Extract widgets
// ❌ Everything in one build()
Widget build(BuildContext context) {
  return Column(children: [
    Text(name),
    ExpensiveWidget(), // rebuilt even when only name changes
  ]);
}

// ✅ Separate widget class
class ExpensiveWidget extends StatelessWidget {
  const ExpensiveWidget();
  @override
  Widget build(BuildContext context) => /* ... */;
}

// 3. Use Selector/Consumer to scope rebuilds
Selector<CartModel, int>(
  selector: (_, cart) => cart.items.length,
  builder: (_, count, __) => Text('$count items'), // only rebuilds when count changes
)

// 4. BlocBuilder with buildWhen
BlocBuilder<UserBloc, UserState>(
  buildWhen: (previous, current) => previous.name != current.name,
  builder: (context, state) => Text(state.name),
)

// 5. Use Keys to prevent unnecessary recreation
```

---

## Q8. What is Impeller and how does it improve Flutter?

**Answer:**
Impeller is Flutter's new rendering engine (replacing Skia for iOS, coming to Android).

**Improvements:**
- **Pre-compiled shaders** — no first-frame jank (Skia compiles shaders at runtime)
- **Predictable performance** — consistent frame times
- **Better Metal/Vulkan utilization** — uses GPU more efficiently
- **Tessellation on CPU** — reduces GPU work for complex paths

```bash
# iOS: Impeller is default since Flutter 3.16
# Android: Opt-in
flutter run --enable-impeller

# Disable (if issues)
flutter run --no-enable-impeller
```

---

## Q9. How do you implement lazy loading and pagination?

**Answer:**

```dart
class _UserListState extends State<UserList> {
  final _scrollController = ScrollController();
  final _users = <User>[];
  int _page = 1;
  bool _isLoading = false;
  bool _hasMore = true;

  @override
  void initState() {
    super.initState();
    _loadMore();
    _scrollController.addListener(() {
      if (_scrollController.position.pixels >=
          _scrollController.position.maxScrollExtent - 200) {
        _loadMore();
      }
    });
  }

  Future<void> _loadMore() async {
    if (_isLoading || !_hasMore) return;
    setState(() => _isLoading = true);

    final newUsers = await api.getUsers(page: _page, limit: 20);
    setState(() {
      _users.addAll(newUsers);
      _page++;
      _isLoading = false;
      _hasMore = newUsers.length == 20;
    });
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      controller: _scrollController,
      itemCount: _users.length + (_hasMore ? 1 : 0),
      itemBuilder: (context, index) {
        if (index == _users.length) return CircularProgressIndicator();
        return UserTile(user: _users[index]);
      },
    );
  }
}
```

---

## Q10. What is the `DevTools` Memory tab and how do you find memory leaks?

**Answer:**

**Common memory leaks in Flutter:**
1. Not disposing `AnimationController`, `TextEditingController`, `ScrollController`
2. Not canceling `StreamSubscription`
3. Not removing `WidgetsBindingObserver`
4. Closures holding references to disposed widgets
5. Global/static references to large objects

```dart
// ✅ Always dispose in dispose()
@override
void dispose() {
  _controller.dispose();
  _subscription.cancel();
  _focusNode.dispose();
  _scrollController.dispose();
  WidgetsBinding.instance.removeObserver(this);
  super.dispose();
}
```

**How to detect:**
1. Open DevTools → Memory tab
2. Take heap snapshot before and after navigation
3. Compare snapshots — look for objects that should be garbage collected
4. Use `dart:developer` `Timeline` events to track allocations
5. Check `flutter doctor` for leak_tracker reports
