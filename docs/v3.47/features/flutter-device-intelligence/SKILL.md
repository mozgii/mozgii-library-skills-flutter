---
name: flutter-device-intelligence
description: Sends device context with sign-ins so a backend can spot unusual activity - a random install ID, device and app facts, network type, a VPN hint, best-effort location, Firebase App Check and a known devices screen, within App Store and Google Play privacy rules. Use when adding new-device alerts, a known devices page, App Check, geo or VPN policy, or when someone asks for a stable device ID on Flutter.
metadata:
  tags:
  - flutter
  - security
  - device-intelligence
  - geolocation
  - vpn
  - app-check
---
# Flutter Device Intelligence

Device context tells your backend how a sign-in looks: a new install, an unusual city, a modified client. The server decides what that means, and the app only reports. Everything here is best effort, so a missing signal never stops a user from signing in.

The rules that keep this legal and useful: location is `whileInUse` only, the server resolves IP addresses, a VPN is never a reason to block on its own, and the install ID is random rather than derived from the hardware.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(device_info_plus|package_info_plus|connectivity_plus|geolocator|firebase_app_check|uuid|flutter_secure_storage|dio):" pubspec.yaml
dart pub outdated --no-dev-dependencies
```

Pins were read from pub.dev on 2026-10-30 for Flutter 3.47.9 and Dart 3.13.7, and the examples were analyzed against them. Everything here was compiled, not run on a device, so test each model and sensor step on real hardware. `references/known-devices-screen.md` imports `material_ui`: on `package:flutter/material.dart`, swap that import, or run `dart fix --apply --code=migrate_design_widgets`.

`connectivity_plus` 7.3.2 needs `dbus` 0.8 through `nm`, while `geolocator` 14.1.2 needs `dbus` 0.7 through `geolocator_linux`, so `connectivity_plus: ^7.3.2` beside `geolocator` fails version solving. `^7.3.1` resolves to 7.3.1 until those two agree. The examples also import `firebase_core: ^5.1.0`, `flutter_riverpod: ^3.4.3`, `material_ui: ^1.9.0`.

## App Store and Play Policy for Device Data

Apple's rule decides how this skill is built. Its privacy page forbids building a unique identifier out of data read from the device, and it names device configuration, location and network connection as examples. Apps found doing this, or using SDKs that do, may be rejected. So don't combine model, OS, network and location into a hash, and don't use the word "fingerprint" for what you ship: send a random ID you generated, with facts the server reads separately.

| Rule | Source | What it means here |
|---|---|---|
| No identifier derived from device properties | Apple user privacy page | Use a random install ID |
| Sharing with a data broker only for fraud or security isn't tracking | Apple user privacy page | The exclusion needs "solely": never reuse this data for ads |
| Persistent IDs (IMEI, IMSI, SIM serial) may not link to personal data | Google Play User Data policy | Never read them |
| FID or a privately stored GUID suits non-ads uses; Play Integrity suits abuse detection | Android user data IDs guide | A random ID, plus App Check |

Declare what you collect: "Device or other IDs" and location in Play's Data Safety form, and matching `NSPrivacyCollectedDataTypes` in the iOS privacy manifest.

## Choosing an Install Identifier

| Option | Choose when | Why the others lose |
|---|---|---|
| Random UUID in secure storage | Default | Yours, and scoped to the app |
| `FirebaseInstallations.instance.getId()` | Firebase is present and you want Google's scoped ID | Android's guide recommends it for most non-ads cases; it adds a dependency |
| `identifierForVendor` | Rarely, and only on iOS | Shared across your apps, and it can be null |
| `AndroidDeviceInfo.id` | Never | It's `Build.ID`, the OS build label: every phone on the same build shares it |

`info.id` from `device_info_plus` isn't ANDROID_ID. The package documents it as a build identifier, such as a changelist number or a tag like `M4-rc20`, so using it as a device ID makes thousands of devices look like one user.

## Worked Example: Device Context on Sign-In

The model carries only what the app knows. City and country come from the IP address on the server, so they aren't here.

```dart
// lib/core/security/device_context.dart
final class DeviceContext {
  const DeviceContext({
    required this.installId,
    required this.platform,
    required this.osVersion,
    required this.appVersion,
    required this.deviceModel,
    required this.networkType,
    required this.vpnHint,
    required this.collectedAt,
    this.latitude,
    this.longitude,
  });

  final String installId;
  final String platform;
  final String osVersion;
  final String appVersion;
  final String deviceModel;
  final String networkType;

  /// True or false only where the OS reports it; null means unknown.
  final bool? vpnHint;
  final double? latitude;
  final double? longitude;
  final DateTime collectedAt;

  Map<String, Object?> toJson() => {
        'install_id': installId,
        'platform': platform,
        'os_version': osVersion,
        'app_version': appVersion,
        'device_model': deviceModel,
        'network_type': networkType,
        'vpn_hint': vpnHint,
        'latitude': latitude,
        'longitude': longitude,
        'collected_at': collectedAt.toIso8601String(),
      };
}
```

### Random Install ID

A random UUID goes into secure storage on first use. On iOS, Keychain items survive an uninstall, so this ID can outlive the app; `first_unlock_this_device` at least keeps it out of backups and off other devices. `references/context-collectors.md` has the `DeviceIdentity` class.

```yaml
dependencies:
  uuid: ^4.6.0
  flutter_secure_storage: ^11.2.0
  device_info_plus: ^13.3.0
  package_info_plus: ^10.2.2
  connectivity_plus: ^7.3.1
  geolocator: ^14.1.2
```

### Network Type and VPN Detection

VPN detection from an app is a hint at best, and on Apple platforms it isn't available. `connectivity_plus` reports `ConnectivityResult.vpn` on Android, but on iOS and macOS a VPN shows up as `other`. Matching interface names doesn't fix that: Apple DTS has confirmed that `utun` interfaces exist with no VPN running, because system services use them, so a `utun` check flags ordinary iPhones. Report `null` on iOS and let the server judge the IP address.

```dart
// lib/core/security/network_context.dart
import 'dart:io';

import 'package:connectivity_plus/connectivity_plus.dart';

({String type, bool? vpnHint}) readNetwork(
  List<ConnectivityResult> results, {
  required bool isAndroid,
}) {
  final type = switch (results) {
    _ when results.contains(ConnectivityResult.wifi) => 'wifi',
    _ when results.contains(ConnectivityResult.mobile) => 'mobile',
    _ when results.contains(ConnectivityResult.ethernet) => 'ethernet',
    _ when results.contains(ConnectivityResult.none) => 'none',
    _ => 'other',
  };
  return (
    type: type,
    vpnHint: isAndroid ? results.contains(ConnectivityResult.vpn) : null,
  );
}

Future<({String type, bool? vpnHint})> networkContext() async {
  final results = await Connectivity().checkConnectivity();
  return readNetwork(results, isAndroid: Platform.isAndroid);
}
```

### Location Only When Permitted

Location needs a reason the user agreed to, and it's never requested during sign-in. This reads a position only if permission already exists. City-level accuracy is enough, so ask for `LocationAccuracy.low`, with a three-second limit, and return null on any failure (`references/context-collectors.md`).

```xml
<!-- ios/Runner/Info.plist -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>Your location helps us detect suspicious sign-ins from unexpected places.</string>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
```

### Assemble DeviceContext Without Blocking Sign-In

Each slow signal has its own short timeout and a default, and the whole call never throws. A single `.timeout` around `Future.wait` that throws would turn a slow GPS fix into a failed login, and a user underground couldn't sign in. A fresh sign-in with partial context is always better than no sign-in.

```dart
// lib/core/security/device_context_service.dart
import 'dart:async';

import 'package:device_info_plus/device_info_plus.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'package:yourapp/core/security/device_context.dart';
import 'package:yourapp/core/security/device_identity.dart';
import 'package:yourapp/core/security/location_context.dart';
import 'package:yourapp/core/security/network_context.dart';

class DeviceContextService {
  DeviceContextService(this._identity);

  factory DeviceContextService.create() => DeviceContextService(
        DeviceIdentity(DeviceInfoPlugin(), const FlutterSecureStorage()),
      );

  final DeviceIdentity _identity;

  static const _budget = Duration(seconds: 2);

  Future<T> _bounded<T>(Future<T> Function() read, T fallback) async {
    try {
      return await read().timeout(_budget);
    } on Object {
      return fallback; // A signal is optional; sign-in is not.
    }
  }

  Future<DeviceContext> collect() async {
    final (installId, device, network, location, package) = await (
      _bounded(_identity.installId, 'unknown'),
      _bounded(
        _identity.describe,
        (platform: 'unknown', model: 'unknown', os: 'unknown'),
      ),
      _bounded(networkContext, (type: 'unknown', vpnHint: null)),
      _bounded(locationIfPermitted, null),
      _bounded<PackageInfo?>(PackageInfo.fromPlatform, null),
    ).wait;

    return DeviceContext(
      installId: installId,
      platform: device.platform,
      osVersion: device.os,
      appVersion: package == null
          ? 'unknown'
          : '${package.version}+${package.buildNumber}',
      deviceModel: device.model,
      networkType: network.type,
      vpnHint: network.vpnHint,
      latitude: location?.lat,
      longitude: location?.lng,
      collectedAt: DateTime.now().toUtc(),
    );
  }
}
```

## Firebase App Check for Your Own API

App Check attests that a request comes from your genuine app. It works only if your server rejects requests with a missing or invalid token. `firebase_app_check` 0.5 removed the `androidProvider` and `appleProvider` parameters, so older snippets don't compile. The defaults are Play Integrity and DeviceCheck. App Attest needs iOS 14 or later, and the example uses the provider that falls back to DeviceCheck.

```yaml
dependencies:
  firebase_app_check: ^0.5.1
  dio: ^5.11.1
```

```dart
// lib/core/network/app_check.dart
import 'package:dio/dio.dart';
import 'package:firebase_app_check/firebase_app_check.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/foundation.dart';

/// Call right after Firebase.initializeApp().
Future<void> activateAppCheck() => FirebaseAppCheck.instance.activate(
      providerAndroid: kDebugMode
          ? const AndroidDebugProvider()
          : const AndroidPlayIntegrityProvider(),
      providerApple: kDebugMode
          ? const AppleDebugProvider()
          : const AppleAppAttestWithDeviceCheckFallbackProvider(),
    );

class AppCheckInterceptor extends Interceptor {
  AppCheckInterceptor(this._apiHost);

  final String _apiHost;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    if (options.uri.host != _apiHost) return handler.next(options);
    try {
      final token = await FirebaseAppCheck.instance.getToken();
      if (token != null) options.headers['X-Firebase-AppCheck'] = token;
    } on FirebaseException {
      // The server's decision: it answers 401 when the header is absent.
    }
    handler.next(options);
  }
}
```

Pass the host of your API's base URL, so a third-party URL called through the same `Dio` never receives the token. On the server, verify with the Admin SDK (`getAppCheck().verifyToken(token)` in Node.js) and return 401 for a missing or invalid token. Firebase products take the token automatically once `activate` finishes.

## Known Devices Screen

`references/known-devices-screen.md` has the complete settings screen, with loading, empty and error states and a sign-out action per device. List model, place and last activity, and keep the install ID off the screen.

## Verify

```dart
// test/core/security/network_context_test.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/core/security/network_context.dart';

void main() {
  test('Android reports a VPN hint', () {
    final network = readNetwork(
      [ConnectivityResult.wifi, ConnectivityResult.vpn],
      isAndroid: true,
    );
    expect(network.type, 'wifi');
    expect(network.vpnHint, isTrue);
  });

  test('iOS reports unknown, never a guess', () {
    final network = readNetwork([ConnectivityResult.other], isAndroid: false);
    expect(network.type, 'other');
    expect(network.vpnHint, isNull);
  });
}
```

On a device, turn on airplane mode, deny location, and sign in: it must still work. Reinstall and confirm what your server sees. In a release build, requests with no App Check header must get a 401.

```bash
grep -rnE "androidInfo\.id|\.identifierForVendor|fingerprint" lib/   # expect nothing
```

## Gotchas

1. **Thousands of users share one device ID.** Symptom: the known devices list merges strangers. Cause: `AndroidDeviceInfo.id` is `Build.ID`. Fix: use a random install ID.
2. **Sign-in fails with `TimeoutException`.** Symptom: logins fail on slow networks. Cause: a timeout that throws around context collection. Fix: give each signal a fallback.
3. **Every iPhone is flagged as a VPN.** Symptom: `maybeVpn` is true everywhere. Cause: a `utun` prefix check. Fix: report null on iOS.
4. **`version solving failed` naming `dbus`.** Symptom: `connectivity_plus` and `geolocator` conflict. Cause: they need different majors. Fix: use `connectivity_plus: ^7.3.1`.
5. **Rejection for fingerprinting.** Symptom: App Review cites device data. Cause: an identifier derived from device properties, or an SDK doing it. Fix: use a random ID, and audit your SDKs.
6. **App Check adds nothing.** Symptom: scripted clients still succeed. Cause: the server never rejects. Fix: enforce it there.
7. **The device name is generic on iOS.** Symptom: every device reads "iPhone". Cause: since iOS 16 the real name needs an entitlement. Fix: show the model instead.

## Anti-Patterns

- Blocking an account on a VPN flag. A corporate VPN user isn't an attacker.
- Resolving the public IP from the device. The server already sees it, and a client value can be forged.
- Asking for location at sign-in. It reads as tracking and lowers trust.
- Treating client context as proof. All of it can be spoofed, so use it only to raise suspicion.
- Logging context in plaintext. Send it over TLS and keep it out of logs.

## Pre-Merge Checklist

- [ ] The install ID is random, with nothing derived from device properties
- [ ] No hash combines device facts
- [ ] `collect()` never throws and has a time budget
- [ ] Location is `whileInUse` and requested only from an explicit user action
- [ ] The VPN hint is null where the OS can't report it
- [ ] The server rejects missing or invalid App Check tokens
- [ ] The debug provider sits behind `kDebugMode`
- [ ] Data Safety and the privacy manifest list what's collected
- [ ] The privacy policy names this data and its security purpose

## See Also

- `flutter-auth` for the sign-in flow this context attaches to
- `flutter-security` for pinning and integrity checks
- `flutter-analytics-firebase` for analytics, which has a separate consent scope
