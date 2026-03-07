# 32 — General: Git Workflow, Agile/Scrum, ASO, Feature Flags, A/B Testing, Payments

---

## Q1. Explain Git branching strategies for mobile development.

**Answer:**

**Git Flow:**
```
main ──────────────────────────────────────────►
  └─ develop ──────────────────────────────────►
       ├─ feature/login ─────┘
       ├─ feature/profile ───┘
       └─ release/1.2.0 ────────┐
                                 └─► main (tag v1.2.0)
  hotfix/crash-fix ──────────────────► main + develop
```

**GitHub Flow (simpler):**
```
main ──────────────────────────────────────────►
  ├─ feature/login ─── PR ──┘
  ├─ fix/crash ─────── PR ──┘
```

**Trunk-Based Development:**
```
main ──────────────────────────────────────────►
  ├─ short-lived branch (1-2 days max) ─┘
  Feature flags control what's released
```

| Strategy | Best For | Complexity |
|----------|----------|------------|
| Git Flow | Large teams, scheduled releases | High |
| GitHub Flow | CI/CD, web-like deployment | Low |
| Trunk-Based | Feature flags, rapid iteration | Medium |

---

## Q2. What are essential Git commands for daily development?

**Answer:**

```bash
# Branch operations
git checkout -b feature/login        # create + switch
git branch -d feature/login          # delete local
git push origin --delete feature/login  # delete remote

# Staging and committing
git add -p                           # stage hunks interactively
git commit --amend                   # modify last commit
git commit --fixup=<sha>             # create fixup commit

# Rebasing
git rebase -i HEAD~5                 # interactive rebase last 5 commits
git rebase main                      # rebase current branch on main
git rebase --autosquash main         # auto-apply fixup commits

# Stashing
git stash push -m "wip: login"      # save work
git stash pop                        # restore latest
git stash list                       # view all stashes

# Cherry-pick
git cherry-pick abc123               # apply specific commit

# Finding bugs
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# Git does binary search to find the bad commit

# Undoing mistakes
git reset --soft HEAD~1              # undo commit, keep changes staged
git reset --mixed HEAD~1             # undo commit, keep changes unstaged
git reset --hard HEAD~1              # undo commit, discard changes
git revert abc123                    # create new commit that undoes abc123

# Log
git log --oneline --graph --all      # visual branch history
git log -p -S "functionName"         # find when code was added/removed
git blame file.dart                  # who changed each line
```

---

## Q3. How do code reviews work and what should you look for?

**Answer:**

**What to check in code reviews:**

1. **Correctness**: Does it do what it claims? Edge cases handled?
2. **Architecture**: Follows project patterns? Right abstractions?
3. **Performance**: N+1 queries? Unnecessary rebuilds? Memory leaks?
4. **Security**: Input validation? Sensitive data exposure? SQL injection?
5. **Testing**: Adequate tests? Edge cases covered?
6. **Naming**: Clear, consistent naming conventions?
7. **Error handling**: Graceful degradation? User-facing errors?
8. **Documentation**: Complex logic explained? API docs?

**PR best practices:**
- Small, focused PRs (< 400 lines)
- Descriptive title and description
- Link to ticket/issue
- Screenshots/videos for UI changes
- Self-review before requesting others
- Respond to all comments before merging

---

## Q4. Explain Agile/Scrum methodology for mobile development.

**Answer:**

**Scrum framework:**
- **Sprint**: 1-2 week iteration
- **Sprint Planning**: Select backlog items, break into tasks, estimate (story points)
- **Daily Standup**: 15 min — what I did, what I'll do, blockers
- **Sprint Review**: Demo to stakeholders
- **Sprint Retrospective**: What went well, what to improve

**Roles:**
- **Product Owner**: Prioritizes backlog, defines requirements
- **Scrum Master**: Facilitates process, removes blockers
- **Dev Team**: Self-organizing, cross-functional

**Estimation:**
- Story points (Fibonacci: 1, 2, 3, 5, 8, 13)
- T-shirt sizing (S, M, L, XL) for initial grooming
- Velocity = average story points per sprint

**Mobile-specific considerations:**
- App Store review time in release planning
- Device/OS fragmentation testing sprints
- Feature flags to decouple deployment from release
- Beta testing (TestFlight/Firebase App Distribution) before release

---

## Q5. What is App Store Optimization (ASO)?

**Answer:**

ASO is optimizing your app listing to rank higher in store search results.

**Key factors:**

| Factor | App Store (iOS) | Play Store (Android) |
|--------|----------------|---------------------|
| **Title** | 30 chars | 30 chars |
| **Subtitle** | 30 chars | N/A |
| **Short description** | N/A | 80 chars |
| **Description** | Not indexed for search | Indexed for search |
| **Keywords** | 100 chars keyword field | From title + description |
| **Screenshots** | Up to 10 | Up to 8 |
| **Ratings** | Major factor | Major factor |
| **Updates** | Frequency matters | Frequency matters |

**Best practices:**
- Research keywords using ASO tools (AppTweak, Sensor Tower)
- A/B test screenshots and icons (Play Store has built-in experiments)
- Respond to reviews (improves rankings)
- Localize listings for top markets
- Encourage ratings with in-app prompts (SKStoreReviewController / In-App Review API)
- First 3 screenshots are most important
- App preview videos increase conversion 20-30%

---

## Q6. How do you implement feature flags in a mobile app?

**Answer:**

```dart
// Using Firebase Remote Config
class FeatureFlags {
  final remoteConfig = FirebaseRemoteConfig.instance;

  Future<void> init() async {
    await remoteConfig.setDefaults({
      'new_checkout_enabled': false,
      'max_upload_size_mb': 10,
      'onboarding_variant': 'control',
    });
    await remoteConfig.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: Duration(seconds: 10),
      minimumFetchInterval: Duration(hours: 1), // 0 for debug
    ));
    await remoteConfig.fetchAndActivate();
  }

  bool get isNewCheckoutEnabled => remoteConfig.getBool('new_checkout_enabled');
  int get maxUploadSize => remoteConfig.getInt('max_upload_size_mb');
  String get onboardingVariant => remoteConfig.getString('onboarding_variant');
}

// Usage
if (featureFlags.isNewCheckoutEnabled) {
  return NewCheckoutScreen();
} else {
  return OldCheckoutScreen();
}

// A/B testing with Firebase
// Create experiment in Firebase Console:
// 1. Define parameter (e.g., 'checkout_button_color')
// 2. Set variants: control='blue', variant_a='green', variant_b='orange'
// 3. Set activation event (e.g., 'purchase_completed')
// 4. Firebase automatically distributes users and measures conversion
```

---

## Q7. How do you integrate payment systems (Stripe/Razorpay)?

**Answer:**

```dart
// Stripe integration in Flutter (stripe_flutter)
class PaymentService {
  Future<void> init() async {
    Stripe.publishableKey = 'pk_live_xxx';
    await Stripe.instance.applySettings();
  }

  Future<void> makePayment(int amountInCents, String currency) async {
    // 1. Create PaymentIntent on YOUR server
    final response = await http.post(
      Uri.parse('$baseUrl/create-payment-intent'),
      body: jsonEncode({'amount': amountInCents, 'currency': currency}),
    );
    final clientSecret = jsonDecode(response.body)['clientSecret'];

    // 2. Initialize payment sheet
    await Stripe.instance.initPaymentSheet(
      paymentSheetParameters: SetupPaymentSheetParameters(
        paymentIntentClientSecret: clientSecret,
        merchantDisplayName: 'My Store',
        style: ThemeMode.system,
      ),
    );

    // 3. Present payment sheet
    await Stripe.instance.presentPaymentSheet();
    // Success!
  }
}

// Server-side (Node.js example)
app.post('/create-payment-intent', async (req, res) => {
  const { amount, currency } = req.body;
  const paymentIntent = await stripe.paymentIntents.create({
    amount, currency,
    automatic_payment_methods: { enabled: true },
  });
  res.json({ clientSecret: paymentIntent.client_secret });
});
```

**Security rules:**
- NEVER process payments client-side only
- Always verify amounts on server
- Use webhooks for payment confirmation
- PCI compliance: use Stripe Elements / Payment Sheet (handles card data)
- Store transaction records on your server

---

## Q8. How do you implement analytics and crash reporting?

**Answer:**

```dart
// Firebase Analytics + Crashlytics

class AnalyticsService {
  final analytics = FirebaseAnalytics.instance;

  // Screen tracking
  void logScreenView(String screenName) {
    analytics.logScreenView(screenName: screenName, screenClass: screenName);
  }

  // Custom events
  void logPurchase(String productId, double amount) {
    analytics.logPurchase(
      currency: 'USD', value: amount,
      items: [AnalyticsEventItem(itemId: productId)],
    );
  }

  // User properties
  void setUserType(String type) {
    analytics.setUserProperty(name: 'user_type', value: type);
  }

  // Custom event
  void logEvent(String name, Map<String, Object> params) {
    analytics.logEvent(name: name, parameters: params);
  }
}

// Crashlytics
class CrashService {
  void init() {
    // Catch Flutter errors
    FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

    // Catch async errors
    PlatformDispatcher.instance.onError = (error, stack) {
      FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
      return true;
    };
  }

  void setUser(String userId) {
    FirebaseCrashlytics.instance.setUserIdentifier(userId);
  }

  void log(String message) {
    FirebaseCrashlytics.instance.log(message);
  }

  void recordError(dynamic error, StackTrace stack) {
    FirebaseCrashlytics.instance.recordError(error, stack);
  }
}
```

**Key metrics to track:**
- DAU/MAU, retention (D1, D7, D30), session duration
- Funnel conversion (onboarding, purchase)
- Crash-free rate (target: >99.5%)
- ANR rate, app start time, frame rendering
- Revenue per user (ARPU), LTV
