---
title: The FCM registration lifecycle
author: MOZGII, Inc.
lang: en
tags: [flutter, fcm, tokens, push, fid]
description: Keeping the server's copy of the device's push address current with Firebase Installation IDs, a listener that can't be skipped, the iOS APNs wait, sign-out cleanup, and upgrading from registration tokens.
---
# The FCM Registration Lifecycle

FCM is moving its push address from the registration token to the Firebase Installation ID (FID). Either way the address can change without warning, and the server's copy has to follow it, or pushes go nowhere and nothing reports an error.

`firebase_messaging` 17 deprecates `getToken`, `onTokenRefresh` and `deleteToken` in favour of `register()`, `onRegistered`, `unregister()` and `onUnregistered`. Firebase's docs say the old methods will be removed and warn against using both sets at once.

## Turning On Firebase Installation IDs

FIDs need a static opt-in on each mobile platform.

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside <application> -->
<meta-data
  android:name="firebase_messaging_installation_id_enabled"
  android:value="true"/>
```

```xml
<!-- ios/Runner/Info.plist -->
<key>FirebaseMessagingInstallationIdEnabled</key>
<true/>
```

## Registering Without Triggering the Permission Prompt

On Android and Apple, FCM registers at startup on its own while auto-init is on, which is the default, and emits the FID through `onRegistered`. Firebase also documents that `register()` asks for notification permission if it hasn't been granted. So calling it at launch would show the prompt on first launch, which is exactly what the skill forbids.

So `start()` works in this order:

1. **Subscribe to `onRegistered` first.** Firebase requires this, and it's what fixes the classic bug: a listener placed after an "unchanged, skip the upload" return only attaches on launches where the value changed, and misses the next rotation.
2. **Stop if permission isn't granted.** Auto-init still delivers the FID through the listener.
3. **Wait for APNs on iOS.** The APNs token must exist before FCM calls, and it isn't guaranteed to have arrived. `_apnsReady` waits a few seconds, and passes straight through on Android.
4. **Call `register()`.** It re-emits the current FID, so a value missed earlier still reaches the server.

Call `start()` again right after the person grants permission. `??=` makes that safe, and an unchanged FID skips the API call, which keeps launches quiet. On web, or with auto-init disabled, `register()` is required, so the same rule applies: call it once permission is granted. On web you can also pass `vapidKey` and `serviceWorkerScriptPath`.

```dart
// lib/core/notifications/token_service.dart
import 'dart:async';
import 'dart:io';

import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

abstract interface class DeviceRegistrationApi {
  Future<void> register({required String fid, required String platform});
  Future<void> unregister(String fid);
}

class FcmRegistrationService {
  FcmRegistrationService(
    this._messaging,
    this._api,
    this._storage, {
    this.isIos = false,
    this.apnsRetry = const Duration(seconds: 1),
  });

  factory FcmRegistrationService.forPlatform(
    FirebaseMessaging messaging,
    DeviceRegistrationApi api,
    FlutterSecureStorage storage,
  ) => FcmRegistrationService(messaging, api, storage, isIos: Platform.isIOS);

  final FirebaseMessaging _messaging;
  final DeviceRegistrationApi _api;
  final FlutterSecureStorage _storage;
  final bool isIos;
  final Duration apnsRetry;

  static const _fidKey = 'fcm_fid';
  StreamSubscription<String>? _registeredSub;
  StreamSubscription<String>? _unregisteredSub;

  Future<void> start() async {
    // Subscribe first: a rotation must reach the server on every launch.
    _registeredSub ??= _messaging.onRegistered.listen(_upload);
    _unregisteredSub ??= _messaging.onUnregistered.listen(_remove);

    // register() would show the permission prompt, so it waits for a grant.
    final settings = await _messaging.getNotificationSettings();
    if (settings.authorizationStatus != AuthorizationStatus.authorized &&
        settings.authorizationStatus != AuthorizationStatus.provisional) {
      return;
    }
    if (!await _apnsReady()) return;
    await _messaging.register();
  }

  Future<bool> _apnsReady() async {
    if (!isIos) return true;
    for (var attempt = 0; attempt < 5; attempt++) {
      if (await _messaging.getAPNSToken() != null) return true;
      await Future<void>.delayed(apnsRetry);
    }
    return false;
  }

  Future<void> _upload(String fid) async {
    if (fid == await _storage.read(key: _fidKey)) return; // No change
    await _api.register(fid: fid, platform: isIos ? 'ios' : 'android');
    await _storage.write(key: _fidKey, value: fid);
  }

  Future<void> _remove(String fid) async {
    if (fid != await _storage.read(key: _fidKey)) return;
    await _api.unregister(fid);
    await _storage.delete(key: _fidKey);
  }

  /// Call on sign-out, so this device stops receiving the account's pushes.
  Future<void> stop() async {
    await _registeredSub?.cancel();
    _registeredSub = null;
    final fid = await _storage.read(key: _fidKey);
    if (fid != null) await _remove(fid);
    await _messaging.unregister();
    await _unregisteredSub?.cancel();
    _unregisteredSub = null;
  }
}
```

## Upgrading From Registration Tokens

Change the client and the server deliberately, because they meet at the send call.

1. **Server first.** The HTTP v1 and Admin SDK send calls take a `fid` target, which Firebase marks as recommended, and mark `token` as deprecated. During the migration period the `token` field also accepts FIDs, so a server that passes the value through keeps working. Firebase gives no removal date.
2. **Add the two flags above**, then replace `getToken` with `onRegistered` and `register()`, `onTokenRefresh` with `onRegistered`, and `deleteToken` with `unregister()`.
3. **Use one set only.** Don't call the token APIs alongside the FID APIs.

The stored key changes too, so the first launch after the upgrade uploads the FID once.

## Registration Tests: A Rotation With an Unchanged FID

The test that matters starts with an FID that matches the stored one, the path where an upload is skipped. A rotation after that must still reach the server. A second test proves the permission rule: without a grant, `register()` is never called. Imports use the package name `app`: replace it with yours.

```dart
// test/core/notifications/token_service_test.dart
import 'dart:async';

import 'package:app/core/notifications/token_service.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';

class FakeSettings extends Fake implements NotificationSettings {
  FakeSettings(this.authorizationStatus);

  @override
  final AuthorizationStatus authorizationStatus;
}

class FakeMessaging extends Fake implements FirebaseMessaging {
  FakeMessaging({
    required this.fid,
    this.status = AuthorizationStatus.authorized,
  });

  String fid;
  AuthorizationStatus status;
  String? apnsToken;
  int registerCalls = 0;
  bool unregistered = false;
  final registrations = StreamController<String>.broadcast(sync: true);
  final removals = StreamController<String>.broadcast(sync: true);

  @override
  Stream<String> get onRegistered => registrations.stream;

  @override
  Stream<String> get onUnregistered => removals.stream;

  @override
  Future<NotificationSettings> getNotificationSettings() async =>
      FakeSettings(status);

  @override
  Future<String?> getAPNSToken() async => apnsToken;

  @override
  Future<void> register({
    String? vapidKey,
    String? serviceWorkerScriptPath,
  }) async {
    registerCalls++;
    registrations.add(fid);
  }

  @override
  Future<void> unregister() async {
    unregistered = true;
    removals.add(fid);
  }
}

class FakeApi implements DeviceRegistrationApi {
  final registered = <String>[];
  final unregistered = <String>[];

  @override
  Future<void> register({
    required String fid,
    required String platform,
  }) async => registered.add(fid);

  @override
  Future<void> unregister(String fid) async => unregistered.add(fid);
}

Future<void> settle() => Future<void>.delayed(const Duration(milliseconds: 5));

void main() {
  const storage = FlutterSecureStorage();

  test('an unchanged FID skips the upload but keeps listening', () async {
    FlutterSecureStorage.setMockInitialValues({'fcm_fid': 'same'});
    final messaging = FakeMessaging(fid: 'same');
    final api = FakeApi();

    await FcmRegistrationService(messaging, api, storage).start();
    await settle();
    expect(api.registered, isEmpty);

    messaging.registrations.add('rotated');
    await settle();
    expect(api.registered, ['rotated']);
    expect(await storage.read(key: 'fcm_fid'), 'rotated');
  });

  test('a changed FID uploads once, however often start runs', () async {
    FlutterSecureStorage.setMockInitialValues({'fcm_fid': 'old'});
    final api = FakeApi();

    final service = FcmRegistrationService(
      FakeMessaging(fid: 'new'),
      api,
      storage,
    );
    await service.start();
    await settle();
    await service.start();
    await settle();

    expect(api.registered, ['new']);
  });

  test('without a permission grant, register is never called', () async {
    FlutterSecureStorage.setMockInitialValues({});
    final messaging = FakeMessaging(
      fid: 'id',
      status: AuthorizationStatus.notDetermined,
    );
    final api = FakeApi();

    await FcmRegistrationService(messaging, api, storage).start();
    expect(messaging.registerCalls, 0);

    // Auto-init still delivers the FID to the listener.
    messaging.registrations.add('auto');
    await settle();
    expect(api.registered, ['auto']);
  });

  test('iOS without an APNs token does not register', () async {
    FlutterSecureStorage.setMockInitialValues({});
    final messaging = FakeMessaging(fid: 'id');

    await FcmRegistrationService(
      messaging,
      FakeApi(),
      storage,
      isIos: true,
      apnsRetry: Duration.zero,
    ).start();

    expect(messaging.registerCalls, 0);
  });

  test('stop removes the FID and stops listening', () async {
    FlutterSecureStorage.setMockInitialValues({'fcm_fid': 'current'});
    final messaging = FakeMessaging(fid: 'current');
    final api = FakeApi();
    final service = FcmRegistrationService(messaging, api, storage);

    await service.start();
    await settle();
    await service.stop();
    messaging.registrations.add('late');
    await settle();

    expect(api.unregistered, ['current']);
    expect(api.registered, isEmpty);
    expect(messaging.unregistered, isTrue);
    expect(await storage.read(key: 'fcm_fid'), isNull);
  });
}
```
