# 08 — Flutter Animations & Advanced UI

---

## Q1. What are implicit animations in Flutter?

**Answer:**
Implicit animations automatically animate between old and new values when a property changes.

```dart
// AnimatedContainer — animates all Container properties
AnimatedContainer(
  duration: Duration(milliseconds: 300),
  curve: Curves.easeInOut,
  width: _expanded ? 200 : 100,
  height: _expanded ? 200 : 100,
  color: _expanded ? Colors.blue : Colors.red,
  child: FlutterLogo(),
)

// AnimatedOpacity
AnimatedOpacity(opacity: _visible ? 1.0 : 0.0, duration: Duration(milliseconds: 500), child: Text('Hello'))

// AnimatedPositioned (inside Stack)
AnimatedPositioned(left: _moved ? 100 : 0, top: _moved ? 100 : 0, duration: Duration(milliseconds: 300), child: Box())

// AnimatedSwitcher — animates between different children
AnimatedSwitcher(
  duration: Duration(milliseconds: 300),
  transitionBuilder: (child, animation) => ScaleTransition(scale: animation, child: child),
  child: Text('$_count', key: ValueKey(_count)), // key change triggers animation
)

// Others: AnimatedAlign, AnimatedPadding, AnimatedDefaultTextStyle, AnimatedCrossFade, AnimatedSize
```

---

## Q2. What are explicit animations in Flutter?

**Answer:**
Explicit animations give you full control over the animation using `AnimationController`.

```dart
class _MyWidgetState extends State<MyWidget> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  late Animation<Color?> _colorAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: Duration(seconds: 1),
      vsync: this, // SingleTickerProviderStateMixin
    );

    _scaleAnimation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.elasticOut),
    );

    _colorAnimation = ColorTween(begin: Colors.red, end: Colors.blue)
      .animate(_controller);

    _controller.forward(); // start
    // _controller.reverse();
    // _controller.repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose(); // MUST dispose
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Transform.scale(
          scale: _scaleAnimation.value,
          child: Container(
            color: _colorAnimation.value,
            child: child, // child is not rebuilt
          ),
        );
      },
      child: Text('Hello'), // static child
    );
  }
}
```

**Key classes:**
- `AnimationController` — drives animation (0.0 → 1.0)
- `Tween` — maps controller value to actual values
- `CurvedAnimation` — applies easing curves
- `AnimatedBuilder` / `AnimatedWidget` — rebuilds UI efficiently

---

## Q3. What is the difference between `SingleTickerProviderStateMixin` and `TickerProviderStateMixin`?

**Answer:**

| Mixin | Use Case |
|-------|----------|
| `SingleTickerProviderStateMixin` | ONE AnimationController |
| `TickerProviderStateMixin` | MULTIPLE AnimationControllers |

```dart
// Single animation
class _State extends State<W> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
}

// Multiple animations
class _State extends State<W> with TickerProviderStateMixin {
  late AnimationController _fadeController;
  late AnimationController _slideController;
}
```

A Ticker provides the timing callbacks that drive animations (linked to `vsync` to avoid off-screen work).

---

## Q4. What is a `Hero` animation?

**Answer:**

```dart
// Source screen
Hero(
  tag: 'user-avatar-${user.id}', // must match on both screens
  child: CircleAvatar(backgroundImage: NetworkImage(user.avatarUrl)),
)

// Destination screen
Hero(
  tag: 'user-avatar-${user.id}',
  child: Image.network(user.avatarUrl, width: 300, height: 300),
)
```

The widget automatically animates between the two screens during navigation. The `tag` must be unique and match on both screens.

---

## Q5. What is `CustomPainter` and how do you use it?

**Answer:**

```dart
class ChartPainter extends CustomPainter {
  final List<double> data;
  ChartPainter(this.data);

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..strokeWidth = 2.0
      ..style = PaintingStyle.stroke;

    // Draw line chart
    final path = Path();
    for (int i = 0; i < data.length; i++) {
      final x = (i / (data.length - 1)) * size.width;
      final y = size.height - (data[i] / 100) * size.height;
      if (i == 0) path.moveTo(x, y); else path.lineTo(x, y);
    }
    canvas.drawPath(path, paint);

    // Draw shapes
    canvas.drawCircle(Offset(50, 50), 30, paint);
    canvas.drawRect(Rect.fromLTWH(100, 100, 50, 50), paint);
    canvas.drawRRect(RRect.fromRectAndRadius(rect, Radius.circular(8)), paint);
  }

  @override
  bool shouldRepaint(ChartPainter oldDelegate) => data != oldDelegate.data;
}

// Usage
CustomPaint(
  painter: ChartPainter([20, 50, 30, 80, 60]),
  size: Size(300, 200),
)
```

---

## Q6. How do you implement staggered animations?

**Answer:**

```dart
class _StaggeredState extends State<Staggered> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _opacity;
  late Animation<Offset> _slide;
  late Animation<double> _scale;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: Duration(milliseconds: 1500),
      vsync: this,
    );

    // Stagger: each animation runs during a portion of the total duration
    _opacity = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _controller, curve: Interval(0.0, 0.3)), // 0-30%
    );
    _slide = Tween<Offset>(begin: Offset(0, 0.5), end: Offset.zero).animate(
      CurvedAnimation(parent: _controller, curve: Interval(0.2, 0.6, curve: Curves.easeOut)), // 20-60%
    );
    _scale = Tween<double>(begin: 0.5, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Interval(0.5, 1.0, curve: Curves.elasticOut)), // 50-100%
    );

    _controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (_, child) {
        return Opacity(
          opacity: _opacity.value,
          child: SlideTransition(
            position: _slide,
            child: Transform.scale(scale: _scale.value, child: child),
          ),
        );
      },
      child: MyWidget(),
    );
  }
}
```

---

## Q7. How do you build responsive/adaptive layouts?

**Answer:**

```dart
class ResponsiveLayout extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth > 1200) {
          return DesktopLayout();
        } else if (constraints.maxWidth > 600) {
          return TabletLayout();
        } else {
          return MobileLayout();
        }
      },
    );
  }
}

// Adaptive widgets (Material on Android, Cupertino on iOS)
import 'dart:io';

Widget adaptiveButton({required VoidCallback onPressed, required String text}) {
  if (Platform.isIOS) {
    return CupertinoButton(onPressed: onPressed, child: Text(text));
  }
  return ElevatedButton(onPressed: onPressed, child: Text(text));
}

// OrientationBuilder
OrientationBuilder(
  builder: (context, orientation) {
    return GridView.count(
      crossAxisCount: orientation == Orientation.portrait ? 2 : 4,
      children: items,
    );
  },
)
```

---

## Q8. Explain `Overlay` and `OverlayEntry`.

**Answer:**

```dart
// Show a custom overlay (tooltip, dropdown, popup)
OverlayEntry? _overlayEntry;

void showOverlay(BuildContext context) {
  final overlay = Overlay.of(context);
  _overlayEntry = OverlayEntry(
    builder: (context) => Positioned(
      top: 100,
      left: 50,
      child: Material(
        elevation: 8,
        child: Container(
          padding: EdgeInsets.all(16),
          child: Text('Custom overlay!'),
        ),
      ),
    ),
  );
  overlay.insert(_overlayEntry!);
}

void hideOverlay() {
  _overlayEntry?.remove();
  _overlayEntry = null;
}
```

---

## Q9. What are `Shader` and `ShaderMask`?

**Answer:**

```dart
// Gradient text
ShaderMask(
  shaderCallback: (Rect bounds) {
    return LinearGradient(
      colors: [Colors.blue, Colors.purple, Colors.red],
    ).createShader(bounds);
  },
  child: Text(
    'Gradient Text',
    style: TextStyle(fontSize: 40, color: Colors.white),
  ),
)

// Fade effect on scrollable
ShaderMask(
  shaderCallback: (Rect bounds) {
    return LinearGradient(
      begin: Alignment.topCenter,
      end: Alignment.bottomCenter,
      colors: [Colors.transparent, Colors.white, Colors.white, Colors.transparent],
      stops: [0.0, 0.1, 0.9, 1.0],
    ).createShader(bounds);
  },
  blendMode: BlendMode.dstIn,
  child: ListView(...),
)
```

---

## Q10. How do you implement drag and drop?

**Answer:**

```dart
// Draggable + DragTarget
Draggable<String>(
  data: 'item_1',
  feedback: Material(
    child: Container(
      padding: EdgeInsets.all(16),
      color: Colors.blue.withOpacity(0.7),
      child: Text('Dragging...'),
    ),
  ),
  childWhenDragging: Opacity(opacity: 0.3, child: ItemWidget()),
  child: ItemWidget(),
)

DragTarget<String>(
  onAcceptWithDetails: (details) {
    setState(() => _droppedItems.add(details.data));
  },
  onWillAcceptWithDetails: (details) => true,
  builder: (context, candidateData, rejectedData) {
    return Container(
      color: candidateData.isNotEmpty ? Colors.green[100] : Colors.grey[200],
      child: Text('Drop here'),
    );
  },
)

// ReorderableListView
ReorderableListView(
  onReorder: (oldIndex, newIndex) {
    setState(() {
      if (newIndex > oldIndex) newIndex--;
      final item = items.removeAt(oldIndex);
      items.insert(newIndex, item);
    });
  },
  children: items.map((item) => ListTile(key: ValueKey(item), title: Text(item))).toList(),
)
```
