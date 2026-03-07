# 04 — Flutter Navigation & Routing

---

## Q1. Explain Navigator 1.0 (imperative) vs Navigator 2.0 (declarative).

**Answer:**

**Navigator 1.0 (Imperative):**
```dart
// Push
Navigator.push(context, MaterialPageRoute(builder: (_) => DetailPage()));
// Pop
Navigator.pop(context);
// Named routes
Navigator.pushNamed(context, '/detail', arguments: {'id': 1});
```

**Navigator 2.0 (Declarative):**
```dart
class MyRouterDelegate extends RouterDelegate<MyRoutePath>
    with ChangeNotifier, PopNavigatorRouterDelegateMixin<MyRoutePath> {
  @override
  Widget build(BuildContext context) {
    return Navigator(
      pages: [
        MaterialPage(child: HomePage()),
        if (_showDetail) MaterialPage(child: DetailPage()),
      ],
      onPopPage: (route, result) {
        _showDetail = false;
        notifyListeners();
        return route.didPop(result);
      },
    );
  }
}
```

| Feature | Navigator 1.0 | Navigator 2.0 |
|---------|--------------|---------------|
| Style | Imperative (push/pop) | Declarative (Pages list) |
| Deep links | Limited | Full support |
| Web URL sync | No | Yes |
| Complexity | Simple | Complex |
| Back button | Automatic | Manual handling |

---

## Q2. Explain GoRouter.

**Answer:**
GoRouter is the recommended routing package that simplifies Navigator 2.0.

```dart
final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
      routes: [
        GoRoute(
          path: 'detail/:id',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return DetailScreen(id: id);
          },
        ),
      ],
    ),
    GoRoute(
      path: '/login',
      builder: (context, state) => LoginScreen(),
    ),
  ],

  // Redirect (e.g., auth guard)
  redirect: (context, state) {
    final isLoggedIn = authService.isLoggedIn;
    final isLoginRoute = state.matchedLocation == '/login';
    if (!isLoggedIn && !isLoginRoute) return '/login';
    if (isLoggedIn && isLoginRoute) return '/';
    return null; // no redirect
  },

  // Error page
  errorBuilder: (context, state) => ErrorScreen(error: state.error),
);

// Use
MaterialApp.router(routerConfig: router)

// Navigate
context.go('/detail/42');       // replace stack
context.push('/detail/42');     // push on stack
context.pop();                  // go back
context.goNamed('detail', pathParameters: {'id': '42'});
```

---

## Q3. How do you implement deep linking in Flutter?

**Answer:**

```dart
// 1. Android: android/app/src/main/AndroidManifest.xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="myapp.com" android:pathPrefix="/product" />
</intent-filter>

// 2. iOS: ios/Runner/Info.plist
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>myapp</string></array>
  </dict>
</array>
// Also: Associated Domains for Universal Links
// applinks:myapp.com

// 3. Handle in GoRouter
GoRoute(
  path: '/product/:id',
  builder: (context, state) {
    return ProductScreen(id: state.pathParameters['id']!);
  },
)
```

**Deep link types:**
- **URI scheme:** `myapp://product/42` (custom, no web fallback)
- **Universal Links (iOS) / App Links (Android):** `https://myapp.com/product/42` (web fallback, requires server config)

---

## Q4. How do you pass data between screens?

**Answer:**

```dart
// 1. Constructor (recommended)
Navigator.push(context, MaterialPageRoute(
  builder: (_) => DetailScreen(user: user),
));

// 2. Named route arguments
Navigator.pushNamed(context, '/detail', arguments: user);
// Receive:
final user = ModalRoute.of(context)!.settings.arguments as User;

// 3. GoRouter path/query parameters
context.go('/detail/42?tab=reviews');
// Receive:
final id = state.pathParameters['id'];
final tab = state.uri.queryParameters['tab'];

// 4. GoRouter extra (complex objects)
context.go('/detail', extra: user);
final user = state.extra as User;

// 5. Return data from screen
final result = await Navigator.push<String>(context, MaterialPageRoute(
  builder: (_) => SelectionScreen(),
));
// In SelectionScreen:
Navigator.pop(context, 'selectedItem');
```

---

## Q5. What are named routes and their limitations?

**Answer:**

```dart
MaterialApp(
  routes: {
    '/': (context) => HomeScreen(),
    '/detail': (context) => DetailScreen(),
    '/settings': (context) => SettingsScreen(),
  },
  onGenerateRoute: (settings) {
    // Dynamic route handling
    if (settings.name?.startsWith('/user/') ?? false) {
      final id = settings.name!.split('/').last;
      return MaterialPageRoute(builder: (_) => UserScreen(id: id));
    }
    return null;
  },
  onUnknownRoute: (settings) {
    return MaterialPageRoute(builder: (_) => NotFoundScreen());
  },
)
```

**Limitations:**
- No type-safe arguments
- No path parameters (`/user/:id`)
- No query parameters
- Hard to handle deep links
- String-based (typo-prone)
- **Solution:** Use GoRouter or auto_route

---

## Q6. What is `auto_route` package?

**Answer:**

```dart
// Define routes with code generation
@AutoRouterConfig()
class AppRouter extends RootStackRouter {
  @override
  List<AutoRoute> get routes => [
    AutoRoute(page: HomeRoute.page, initial: true),
    AutoRoute(page: DetailRoute.page, path: '/detail/:id'),
    AutoRoute(
      page: DashboardRoute.page,
      children: [
        AutoRoute(page: ProfileRoute.page),
        AutoRoute(page: SettingsRoute.page),
      ],
    ),
  ];
}

// Navigate (type-safe)
context.router.push(DetailRoute(id: 42));
context.router.replace(HomeRoute());
context.router.pop();

// Guards
class AuthGuard extends AutoRouteGuard {
  @override
  void onNavigation(NavigationResolver resolver, StackRouter router) {
    if (isAuthenticated) {
      resolver.next(true);
    } else {
      router.push(LoginRoute(onResult: (success) {
        resolver.next(success);
      }));
    }
  }
}
```

---

## Q7. How do you handle nested navigation (bottom tabs)?

**Answer:**

```dart
// GoRouter with ShellRoute
final router = GoRouter(
  routes: [
    ShellRoute(
      builder: (context, state, child) {
        return ScaffoldWithBottomNav(child: child);
      },
      routes: [
        GoRoute(path: '/home', builder: (_, __) => HomeScreen()),
        GoRoute(path: '/search', builder: (_, __) => SearchScreen()),
        GoRoute(path: '/profile', builder: (_, __) => ProfileScreen()),
      ],
    ),
  ],
);

// StatefulShellRoute (preserves state per tab)
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) {
    return ScaffoldWithNavBar(navigationShell: navigationShell);
  },
  branches: [
    StatefulShellBranch(routes: [
      GoRoute(path: '/home', builder: (_, __) => HomeScreen()),
    ]),
    StatefulShellBranch(routes: [
      GoRoute(path: '/search', builder: (_, __) => SearchScreen()),
    ]),
  ],
)
```

---

## Q8. What are route transitions and how do you customize them?

**Answer:**

```dart
// Custom page transition
GoRoute(
  path: '/detail',
  pageBuilder: (context, state) {
    return CustomTransitionPage(
      child: DetailScreen(),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return SlideTransition(
          position: Tween<Offset>(
            begin: Offset(1, 0),
            end: Offset.zero,
          ).animate(CurvedAnimation(
            parent: animation,
            curve: Curves.easeInOut,
          )),
          child: child,
        );
      },
    );
  },
)

// MaterialPageRoute (slide up on Android, slide right on iOS)
// CupertinoPageRoute (always slide right)
// FadeTransition, ScaleTransition, RotationTransition
```

---

## Q9. How do you implement route guards / authentication flow?

**Answer:**

```dart
// GoRouter redirect
GoRouter(
  redirect: (context, state) {
    final isLoggedIn = ref.read(authProvider).isLoggedIn;
    final isLoggingIn = state.matchedLocation == '/login';
    final isOnboarding = state.matchedLocation == '/onboarding';

    // Not logged in → redirect to login
    if (!isLoggedIn && !isLoggingIn) return '/login';

    // Logged in but on login page → redirect to home
    if (isLoggedIn && isLoggingIn) return '/';

    return null; // no redirect
  },
  refreshListenable: authNotifier, // re-evaluate redirect when auth changes
)

// Per-route redirect
GoRoute(
  path: '/admin',
  redirect: (context, state) {
    if (!isAdmin) return '/unauthorized';
    return null;
  },
  builder: (_, __) => AdminScreen(),
)
```

---

## Q10. What is `WillPopScope` / `PopScope`?

**Answer:**

```dart
// PopScope (Flutter 3.16+, replaces WillPopScope)
PopScope(
  canPop: false, // prevents system back
  onPopInvokedWithResult: (didPop, result) {
    if (didPop) return;
    // Show confirmation dialog
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text('Discard changes?'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text('Cancel')),
          TextButton(
            onPressed: () {
              Navigator.pop(context); // close dialog
              Navigator.pop(context); // pop page
            },
            child: Text('Discard'),
          ),
        ],
      ),
    );
  },
  child: EditForm(),
)
```
