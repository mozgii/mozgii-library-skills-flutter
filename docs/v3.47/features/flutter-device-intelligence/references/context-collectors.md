---
title: "Flutter device context collectors: install ID and location"
author: MOZGII, Inc.
lang: en
tags: [flutter, security, device-intelligence, geolocation]
description: The DeviceIdentity class with a random install ID and device facts, and a location reader that never prompts and never throws.
---
# Device Context Collectors

## Random Install ID and Device Facts

`DeviceIdentity` creates a random UUID once and reads the model and OS version. On iOS it uses `modelName` rather than `name`: since iOS 16 `name` returns a generic value without an entitlement, and it's user data anyway.

```dart
// lib/core/security/device_identity.dart
import 'dart:io';

import 'package:device_info_plus/device_info_plus.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:uuid/uuid.dart';

class DeviceIdentity {
  DeviceIdentity(this._deviceInfo, this._storage);

  final DeviceInfoPlugin _deviceInfo;
  final FlutterSecureStorage _storage;

  static const _key = 'install_id';
  static const _options = IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
  );

  Future<String> installId() async {
    final existing = await _storage.read(key: _key, iOptions: _options);
    if (existing != null) return existing;
    final created = const Uuid().v4();
    await _storage.write(key: _key, value: created, iOptions: _options);
    return created;
  }

  Future<({String platform, String model, String os})> describe() async {
    if (Platform.isIOS) {
      final info = await _deviceInfo.iosInfo;
      return (
        platform: 'ios',
        model: info.modelName,
        os: info.systemVersion,
      );
    }
    final info = await _deviceInfo.androidInfo;
    return (
      platform: 'android',
      model: '${info.manufacturer} ${info.model}',
      os: 'Android ${info.version.release} (API ${info.version.sdkInt})',
    );
  }
}
```

## Location Reader That Never Prompts

`locationIfPermitted` reads a position only when permission already exists. A prompt during sign-in reads as tracking, so `requestLocationPermission` belongs behind a setting the user turns on. On Android, `ACCESS_COARSE_LOCATION` alone gives roughly city-block accuracy and uses network services, so a first fix can take a while on some devices; the time limit covers that.

```dart
// lib/core/security/location_context.dart
import 'dart:async';

import 'package:geolocator/geolocator.dart';

Future<({double lat, double lng})?> locationIfPermitted() async {
  final permission = await Geolocator.checkPermission();
  if (permission != LocationPermission.whileInUse &&
      permission != LocationPermission.always) {
    return null; // Never prompt here: request only from a settings toggle.
  }
  try {
    final position = await Geolocator.getCurrentPosition(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.low,
        timeLimit: Duration(seconds: 3),
      ),
    );
    return (lat: position.latitude, lng: position.longitude);
  } on TimeoutException {
    return null;
  } on LocationServiceDisabledException {
    return null;
  } on PermissionDeniedException {
    return null;
  }
}

/// Call this from the switch that turns on location-based security.
Future<bool> requestLocationPermission() async {
  final permission = await Geolocator.requestPermission();
  return permission == LocationPermission.whileInUse;
}
```
