# 09 — Flutter Platform Channels & Native Integration

---

## Q1. What are Platform Channels? Explain the types.

**Answer:**
Platform Channels enable communication between Dart and native code (Kotlin/Swift).

| Channel Type | Communication | Use Case |
|-------------|---------------|----------|
| **MethodChannel** | Request-response (one-time) | Call native API, get battery level |
| **EventChannel** | Stream of events (continuous) | Sensor data, location updates |
| **BasicMessageChannel** | Custom codec messages | Raw message passing |

---

## Q2. How do you implement a MethodChannel?

**Answer:**

```dart
// Dart side
class BatteryService {
  static const _channel = MethodChannel('com.app/battery');

  Future<int> getBatteryLevel() async {
    try {
      final level = await _channel.invokeMethod<int>('getBatteryLevel');
      return level ?? -1;
    } on PlatformException catch (e) {
      throw Exception('Failed: ${e.message}');
    }
  }
}
```

```kotlin
// Android (Kotlin) — MainActivity.kt
class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.app/battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val level = getBatteryLevel()
                        if (level != -1) result.success(level)
                        else result.error("UNAVAILABLE", "Battery level not available", null)
                    }
                    else -> result.notImplemented()
                }
            }
    }

    private fun getBatteryLevel(): Int {
        val batteryManager = getSystemService(BATTERY_SERVICE) as BatteryManager
        return batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
    }
}
```

```swift
// iOS (Swift) — AppDelegate.swift
@UIApplicationMain
class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        let controller = window?.rootViewController as! FlutterViewController
        let channel = FlutterMethodChannel(name: "com.app/battery", binaryMessenger: controller.binaryMessenger)

        channel.setMethodCallHandler { (call, result) in
            if call.method == "getBatteryLevel" {
                let device = UIDevice.current
                device.isBatteryMonitoringEnabled = true
                result(Int(device.batteryLevel * 100))
            } else {
                result(FlutterMethodNotImplemented)
            }
        }
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}
```

---

## Q3. How do you implement an EventChannel?

**Answer:**

```dart
// Dart side
class SensorService {
  static const _channel = EventChannel('com.app/accelerometer');

  Stream<Map<String, double>> get accelerometerEvents {
    return _channel.receiveBroadcastStream().map((event) {
      final map = Map<String, dynamic>.from(event);
      return {
        'x': (map['x'] as num).toDouble(),
        'y': (map['y'] as num).toDouble(),
        'z': (map['z'] as num).toDouble(),
      };
    });
  }
}

// Usage
SensorService().accelerometerEvents.listen((data) {
  print('x: ${data['x']}, y: ${data['y']}, z: ${data['z']}');
});
```

```kotlin
// Android (Kotlin)
class SensorStreamHandler(private val context: Context) : EventChannel.StreamHandler {
    private var sensorManager: SensorManager? = null
    private var listener: SensorEventListener? = null

    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        val accelerometer = sensorManager?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)

        listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                event?.let {
                    events?.success(mapOf("x" to it.values[0], "y" to it.values[1], "z" to it.values[2]))
                }
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        sensorManager?.registerListener(listener, accelerometer, SensorManager.SENSOR_DELAY_NORMAL)
    }

    override fun onCancel(arguments: Any?) {
        sensorManager?.unregisterListener(listener)
    }
}
```

---

## Q4. What is Pigeon and why use it?

**Answer:**
Pigeon is a code generator that creates type-safe platform channel bindings — eliminates manual channel boilerplate.

```dart
// Define API in Dart
@ConfigurePigeon(PigeonOptions(dartOut: 'lib/src/messages.g.dart'))
@HostApi()
abstract class UserApi {
  @async
  User getUser(int id);
  void saveUser(User user);
}

class User {
  final int id;
  final String name;
  final String email;
  User({required this.id, required this.name, required this.email});
}

// Run: dart run pigeon --input pigeons/messages.dart
// Generates:
//   - Dart bindings
//   - Kotlin/Java code
//   - Swift/ObjC code
```

**Benefits:** Type safety, no string-based method names, auto-generated serialization, compile-time errors.

---

## Q5. How do you embed native views in Flutter (Platform Views)?

**Answer:**

```dart
// Android — embed native Android view in Flutter
// Dart side
Widget build(BuildContext context) {
  return AndroidView(
    viewType: 'native-map-view',
    creationParams: {'lat': 37.7749, 'lng': -122.4194},
    creationParamsCodec: StandardMessageCodec(),
  );
}

// iOS — embed native iOS view in Flutter
Widget build(BuildContext context) {
  return UiKitView(
    viewType: 'native-map-view',
    creationParams: {'lat': 37.7749, 'lng': -122.4194},
    creationParamsCodec: StandardMessageCodec(),
  );
}

// Platform-agnostic
Widget build(BuildContext context) {
  // HybridComposition (slower but more compatible) or Virtual Display (faster)
  return PlatformViewLink(
    viewType: 'native-map-view',
    surfaceFactory: (context, controller) => AndroidViewSurface(
      controller: controller as AndroidViewController,
      gestureRecognizers: <Factory<OneSequenceGestureRecognizer>>{},
      hitTestBehavior: PlatformViewHitTestBehavior.opaque,
    ),
    onCreatePlatformView: (params) => PlatformViewsService.initSurfaceAndroidView(
      id: params.id,
      viewType: 'native-map-view',
      layoutDirection: TextDirection.ltr,
    )..create(),
  );
}
```

---

## Q6. What is FFI (Foreign Function Interface) in Dart?

**Answer:**
`dart:ffi` allows Dart to call native C libraries directly without platform channels.

```dart
// Load native library
import 'dart:ffi';

typedef NativeAdd = Int32 Function(Int32, Int32);
typedef DartAdd = int Function(int, int);

void main() {
  final dylib = DynamicLibrary.open('libnative.so'); // or .dylib / .dll
  final add = dylib.lookupFunction<NativeAdd, DartAdd>('add');
  print(add(3, 4)); // 7
}
```

```c
// native_code.c
int add(int a, int b) { return a + b; }
```

**Use cases:** Image processing, crypto, audio/video codecs, legacy C/C++ libraries.
**Advantage over platform channels:** No serialization overhead, synchronous calls.

---

## Q7. What is `federated plugin` architecture?

**Answer:**

```
my_plugin/                     # App-facing package
├── lib/my_plugin.dart        # API that apps use
my_plugin_platform_interface/  # Platform interface package
├── lib/platform_interface.dart  # Abstract class
my_plugin_android/             # Android implementation
├── lib/my_plugin_android.dart
my_plugin_ios/                 # iOS implementation
├── lib/my_plugin_ios.dart
my_plugin_web/                 # Web implementation
├── lib/my_plugin_web.dart
```

**Benefits:**
- Each platform can be developed independently
- Third parties can add platform support
- App only includes needed platforms
- Used by official plugins (camera, url_launcher, etc.)
