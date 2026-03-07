# 26 — Cross-Platform & General Mobile Concepts

---

## Q1. What is the difference between native, hybrid, and cross-platform development?

**Answer:**

| Approach | Languages | UI Rendering | Performance | Examples |
|----------|-----------|-------------|-------------|----------|
| **Native** | Swift/Kotlin | Platform widgets | Best | Standard iOS/Android |
| **Cross-platform** | Dart/Kotlin | Own engine or native widgets | Near-native | Flutter, KMP, React Native |
| **Hybrid** | HTML/CSS/JS | WebView | Slowest | Ionic, Cordova |
| **PWA** | HTML/CSS/JS | Browser | Limited | Web apps |

---

## Q2. Explain REST API design principles.

**Answer:**

```
GET    /api/v1/users          → List users
GET    /api/v1/users/42       → Get user 42
POST   /api/v1/users          → Create user
PUT    /api/v1/users/42       → Replace user 42
PATCH  /api/v1/users/42       → Partial update user 42
DELETE /api/v1/users/42       → Delete user 42
```

**HTTP Status Codes:**

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content (successful delete) |
| 301 | Moved Permanently |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (not authorized) |
| 404 | Not Found |
| 409 | Conflict (duplicate) |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error |
| 503 | Service Unavailable |

**Best practices:**
- Use nouns, not verbs (`/users` not `/getUsers`)
- Use plural nouns (`/users` not `/user`)
- Nest resources (`/users/42/posts`)
- Use query params for filtering (`/users?role=admin&page=2`)
- Version your API (`/v1/users`)
- Use HATEOAS for discoverability

---

## Q3. Explain OAuth 2.0 and JWT.

**Answer:**

**OAuth 2.0 Flow (Authorization Code with PKCE for mobile):**
```
1. App generates code_verifier (random string) and code_challenge (SHA256 hash)
2. App opens browser → Authorization Server login page
   GET /authorize?client_id=X&redirect_uri=Y&code_challenge=Z&code_challenge_method=S256
3. User logs in → Authorization Server redirects to app with authorization_code
4. App exchanges code for tokens:
   POST /token { code, code_verifier, redirect_uri }
5. Server returns: { access_token, refresh_token, expires_in }
6. App uses access_token in API requests: Authorization: Bearer <token>
7. When access_token expires, use refresh_token to get new one
```

**JWT (JSON Web Token):**
```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0Mn0.signature

// Header: { "alg": "HS256", "typ": "JWT" }
// Payload: { "user_id": 42, "exp": 1709733600, "iss": "myapp.com" }
// Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

**Security best practices:**
- Store tokens in secure storage (Keychain / EncryptedSharedPrefs)
- Use short-lived access tokens (15-30 min)
- Implement token refresh
- Use PKCE for mobile OAuth
- Never store tokens in plain text

---

## Q4. What are mobile security best practices?

**Answer:**

| Area | Practice |
|------|----------|
| **Storage** | Keychain (iOS) / EncryptedSharedPrefs (Android) for sensitive data |
| **Network** | HTTPS only, certificate pinning, no sensitive data in URLs |
| **Authentication** | Biometrics, OAuth 2.0 + PKCE, MFA |
| **Code** | Obfuscation, tamper detection, no hardcoded secrets |
| **Input** | Validate all user input, prevent SQL injection, XSS |
| **Logging** | No sensitive data in logs, disable in production |
| **Dependencies** | Keep updated, audit for vulnerabilities |
| **Clipboard** | Clear sensitive data from clipboard |
| **Screenshots** | Blur/hide sensitive screens in app switcher |
| **Root/Jailbreak** | Detect and warn or restrict |

```dart
// Flutter: Certificate pinning with Dio
dio.httpClientAdapter = IOHttpClientAdapter(
  createHttpClient: () {
    final client = HttpClient();
    client.badCertificateCallback = (cert, host, port) {
      // Verify certificate fingerprint
      return cert.sha256 == expectedFingerprint;
    };
    return client;
  },
);
```

---

## Q5. Explain mobile app accessibility.

**Answer:**

```dart
// Flutter accessibility
Semantics(
  label: 'Delete item',
  hint: 'Double tap to delete this item',
  button: true,
  child: IconButton(
    icon: Icon(Icons.delete),
    onPressed: onDelete,
  ),
)

// Exclude from semantics
ExcludeSemantics(child: DecorativeImage())

// Merge semantics
MergeSemantics(
  child: Row(
    children: [
      Icon(Icons.star),
      Text('4.5 stars'),
    ],
  ),
)
```

```swift
// iOS accessibility
button.accessibilityLabel = "Delete item"
button.accessibilityHint = "Double tap to delete"
button.accessibilityTraits = .button
image.isAccessibilityElement = false // decorative

// SwiftUI
Image("logo")
    .accessibilityLabel("Company logo")
    .accessibilityHint("Decorative image")
```

**Checklist:**
- Semantic labels for all interactive elements
- Sufficient color contrast (4.5:1 for text)
- Support Dynamic Type / text scaling
- Support screen readers (VoiceOver / TalkBack)
- Touch targets ≥ 44×44 points (iOS) / 48×48 dp (Android)
- Don't rely solely on color to convey information

---

## Q6. What is Internationalization (i18n) and Localization (l10n)?

**Answer:**

```dart
// Flutter localization (flutter_localizations + intl)
// l10n.yaml
// arb-dir: lib/l10n
// template-arb-file: app_en.arb

// lib/l10n/app_en.arb
{
  "@@locale": "en",
  "hello": "Hello {name}",
  "@hello": { "placeholders": { "name": { "type": "String" } } },
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "price": "Price: {amount}",
  "@price": { "placeholders": { "amount": { "type": "double", "format": "currency", "optionalParameters": { "symbol": "$" } } } }
}

// lib/l10n/app_hi.arb
{
  "@@locale": "hi",
  "hello": "नमस्ते {name}",
  "itemCount": "{count, plural, =0{कोई आइटम नहीं} =1{1 आइटम} other{{count} आइटम}}"
}

// Usage
Text(AppLocalizations.of(context)!.hello('Alice'))
Text(AppLocalizations.of(context)!.itemCount(5))
```

**Considerations:** RTL languages, date/time/currency formatting, pluralization rules, string externalization.

---

## Q7. Explain push notification architecture.

**Answer:**

```
App → Register → FCM/APNs → Returns token → App sends token to backend

Backend → Sends notification payload → FCM/APNs → Delivers to device → App handles
```

| Platform | Service | Payload limit |
|----------|---------|--------------|
| iOS | APNs (Apple Push Notification Service) | 4KB |
| Android | FCM (Firebase Cloud Messaging) | 4KB |

**Notification types:**
- **Alert/Notification** — system displays it
- **Data/Silent** — app processes in background
- **Rich** — images, actions, custom UI

---

## Q8. What is CI/CD and why is it important for mobile?

**Answer:**

**CI (Continuous Integration):**
- Automatically build and test on every commit/PR
- Catch bugs early
- Ensure code quality (lint, format, test coverage)

**CD (Continuous Delivery/Deployment):**
- Automatically deploy to TestFlight / Internal Testing
- Optionally auto-deploy to production

**Mobile-specific challenges:**
- Code signing complexity (certificates, provisioning profiles)
- App store review process (not instant deployment)
- Multiple platforms (iOS + Android)
- Device fragmentation (testing on real devices)
- Longer build times (especially iOS)

---

## Q9. What are design patterns specific to mobile?

**Answer:**

| Pattern | Description | Example |
|---------|-------------|---------|
| **Coordinator** | Separates navigation from view controllers | iOS navigation |
| **Repository** | Abstracts data sources | API + DB behind single interface |
| **Use Case / Interactor** | Encapsulates single business action | `LoginUseCase`, `FetchUsersUseCase` |
| **Adapter** | Transforms data between layers | API model → domain model |
| **Facade** | Simplifies complex subsystem | `NetworkManager` wrapping URLSession |
| **Memento** | Saves and restores state | `onSaveInstanceState` / state restoration |
| **Pub/Sub** | Event-driven communication | `NotificationCenter`, BLoC events |

---

## Q10. Explain mobile app analytics and crash reporting.

**Answer:**

**Common tools:**
- **Firebase Analytics** — events, user properties, funnels
- **Firebase Crashlytics** — crash reports, stack traces
- **Sentry** — error tracking, performance monitoring
- **Mixpanel** — user behavior analytics
- **Amplitude** — product analytics

```dart
// Firebase Analytics (Flutter)
await FirebaseAnalytics.instance.logEvent(
  name: 'purchase',
  parameters: {
    'item_id': 'SKU_123',
    'item_name': 'Premium Plan',
    'value': 9.99,
    'currency': 'USD',
  },
);

// Firebase Crashlytics
FirebaseCrashlytics.instance.recordError(error, stackTrace, reason: 'API call failed');
FirebaseCrashlytics.instance.setCustomKey('user_id', '42');
FirebaseCrashlytics.instance.log('User tapped checkout button');
```
