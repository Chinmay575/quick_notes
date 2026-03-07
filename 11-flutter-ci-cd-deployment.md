# 11 — Flutter CI/CD & Deployment

---

## Q1. How do you set up CI/CD for a Flutter project?

**Answer:**

```yaml
# .github/workflows/flutter.yml (GitHub Actions)
name: Flutter CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: 'stable'
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test --coverage
      - uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info

  build-android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build appbundle --release
      - uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: build/app/outputs/bundle/release/

  build-ios:
    needs: test
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build ipa --release --no-codesign
```

---

## Q2. How do you manage app signing?

**Answer:**

**Android:**
```properties
# android/key.properties (DO NOT commit)
storePassword=password
keyPassword=password
keyAlias=upload
storeFile=../upload-keystore.jks
```

```groovy
// android/app/build.gradle
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

**iOS:** Use Xcode Automatic Signing or Fastlane Match for CI/CD.

---

## Q3. What is Fastlane and how do you use it with Flutter?

**Answer:**

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  lane :beta do
    build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
    upload_to_testflight
  end

  lane :release do
    build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
    upload_to_app_store(skip_metadata: true, skip_screenshots: true)
  end
end

# android/fastlane/Fastfile
platform :android do
  lane :beta do
    upload_to_play_store(track: 'internal', aab: '../build/app/outputs/bundle/release/app-release.aab')
  end

  lane :release do
    upload_to_play_store(track: 'production', aab: '../build/app/outputs/bundle/release/app-release.aab')
  end
end
```

```bash
# Run
cd ios && fastlane beta
cd android && fastlane release
```

---

## Q4. How do you manage different environments (dev, staging, prod)?

**Answer:**

```dart
// Using --dart-define
// flutter run --dart-define=ENV=dev --dart-define=API_URL=https://dev.api.com

class AppConfig {
  static const String env = String.fromEnvironment('ENV', defaultValue: 'dev');
  static const String apiUrl = String.fromEnvironment('API_URL');
  static const bool isProduction = env == 'prod';
}

// Using flavor (Android productFlavors + iOS schemes)
// flutter run --flavor dev -t lib/main_dev.dart
// flutter run --flavor prod -t lib/main_prod.dart

// main_dev.dart
void main() {
  AppConfig.init(Environment.dev);
  runApp(MyApp());
}

// main_prod.dart
void main() {
  AppConfig.init(Environment.prod);
  runApp(MyApp());
}
```

---

## Q5. How do you manage versioning and build numbers?

**Answer:**

```yaml
# pubspec.yaml
version: 1.2.3+45
# 1.2.3 = version name (shown to users)
# 45    = build number (internal, must increment)
```

```bash
# Override at build time
flutter build apk --build-name=1.2.3 --build-number=45

# Auto-increment in CI
BUILD_NUMBER=$(($(git rev-list --count HEAD)))
flutter build apk --build-number=$BUILD_NUMBER
```

---

## Q6. What is Codemagic?

**Answer:**
Codemagic is a CI/CD platform designed specifically for Flutter/Dart.

**Key features:**
- Pre-configured Flutter build machines (macOS for iOS builds)
- Automatic code signing
- Publish directly to App Store Connect and Google Play
- Build triggers on push/PR
- Environment variables and secrets management
- Integration testing on real devices

```yaml
# codemagic.yaml
workflows:
  release:
    name: Release
    triggering:
      events: [push]
      branch_patterns: [{pattern: main, include: true}]
    scripts:
      - name: Build
        script: |
          flutter build appbundle --release
          flutter build ipa --release
    publishing:
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
      app_store_connect:
        api_key: $APP_STORE_CONNECT_API_KEY
```

---

## Q7. How do you handle app updates (force update, in-app update)?

**Answer:**

```dart
// Remote Config approach
class UpdateChecker {
  Future<UpdateInfo> checkForUpdate() async {
    final config = await FirebaseRemoteConfig.instance.fetchAndActivate();
    final minVersion = FirebaseRemoteConfig.instance.getString('min_version');
    final latestVersion = FirebaseRemoteConfig.instance.getString('latest_version');
    final currentVersion = (await PackageInfo.fromPlatform()).version;

    if (Version.parse(currentVersion) < Version.parse(minVersion)) {
      return UpdateInfo.forceUpdate;
    }
    if (Version.parse(currentVersion) < Version.parse(latestVersion)) {
      return UpdateInfo.optionalUpdate;
    }
    return UpdateInfo.upToDate;
  }
}

// Android In-App Update (using in_app_update package)
final updateInfo = await InAppUpdate.checkForUpdate();
if (updateInfo.updateAvailability == UpdateAvailability.updateAvailable) {
  await InAppUpdate.performImmediateUpdate(); // or startFlexibleUpdate()
}
```

---

## Q8. How do you implement Firebase distribution for testing?

**Answer:**

```bash
# Install Firebase CLI
firebase login
firebase init

# Distribute Android
flutter build apk --release
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-release.apk \
  --app YOUR_FIREBASE_APP_ID \
  --groups "testers" \
  --release-notes "Bug fixes and improvements"

# Distribute iOS
flutter build ipa --release
firebase appdistribution:distribute build/ios/ipa/MyApp.ipa \
  --app YOUR_FIREBASE_IOS_APP_ID \
  --groups "testers"
```

---

## Q9. What are obfuscation and ProGuard/R8?

**Answer:**

```bash
# Dart code obfuscation
flutter build apk --obfuscate --split-debug-info=build/debug-info

# This:
# 1. Renames classes/methods to short meaningless names
# 2. Saves mapping in debug-info/ for crash report symbolication
```

**ProGuard/R8 (Android native code):**
```groovy
// android/app/build.gradle
buildTypes {
    release {
        minifyEnabled true      // enable R8
        shrinkResources true    // remove unused resources
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

```
# proguard-rules.pro
-keep class io.flutter.** { *; }
-keep class com.google.firebase.** { *; }
-dontwarn com.google.android.play.core.**
```
