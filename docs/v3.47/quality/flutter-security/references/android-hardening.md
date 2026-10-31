---
title: "Flutter Android hardening: network config, R8, recents and biometric keys"
author: MOZGII, Inc.
lang: en
tags: [flutter, android, security, r8, network-security-config, biometrics]
description: Android-specific hardening for a Flutter release - Network Security Configuration, pin format, R8 keep rules, excludeFromRecents and biometric-bound storage. Use when locking down the Android side of an app.
---
# Android Hardening for Flutter

## Android Network Security Configuration

Network Security Configuration blocks cleartext traffic and can pin certificates for platform networking, declared in XML. Cleartext is already off by default from API 28, so stating it protects you from a dependency or a flavor turning it back on. The XML declaration has to be the first thing in the file, so the path comment goes below it.

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>

    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- Backup pin: a chain is valid when it contains any pinned key. -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

Android's page describes this config for secure connections and doesn't list which HTTP stacks honor it. `dart:io`'s `HttpClient` opens its own sockets, so don't count these pins as covering Dart traffic: keep the dio pinning from the skill, and prove each stack behind a proxy.

`expiration` turns pinning off after that date, so an app that nobody updated keeps working. The page warns that the same date lets an attacker wait it out. Without the attribute, the pins never expire.

### Pin format for pin-set

The value in `<pin digest="SHA-256">` is the base64 SHA-256 of the certificate's SubjectPublicKeyInfo (SPKI), not of the whole certificate. That differs from the leaf fingerprint the dio example uses.

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | base64
```

### Debug-only cleartext and local proxies

Use `<debug-overrides>` for anything that must work only in a debug build. Android ignores the section when `android:debuggable` is false, and stores don't accept debuggable apps. Certificates listed there default to `overridePins="true"`, so pinning is skipped for chains using them.

```xml
<debug-overrides>
    <trust-anchors>
        <certificates src="user"/>
    </trust-anchors>
</debug-overrides>
```

To allow cleartext to a local server in debug only, add a resource with the same name in the debug source set, `android/app/src/debug/res/xml/network_security_config.xml`. A different file name would not override the release file: Android merges resources by name per build variant.

## R8 and Keep Rules

R8 is on by default for release builds. The Flutter Gradle plugin sets `isMinifyEnabled = true` and `isShrinkResources = true` on the release build type unless you pass `--no-shrink`, and it adds `proguard-android-optimize.txt`, its own `flutter_proguard_rules.pro`, and `android/app/proguard-rules.pro` if that file exists. So you don't have to enable anything in Gradle. Add a keep rule only when a release build fails or crashes on a stripped class.

```proguard
# android/app/proguard-rules.pro
-keepattributes *Annotation*
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
# flutter_stripe, when you use it
-dontwarn com.stripe.android.pushProvisioning.**
-keep class com.stripe.** { *; }
```

Flutter's own rules already cover its embedding and `FlutterPlugin` implementations, so older blanket `-keep class io.flutter.**` lines aren't needed. The old `androidx.security.crypto` rule is not needed either: `flutter_secure_storage` 11 removed `encryptedSharedPreferences`.

If you set the block yourself, Kotlin DSL is what new projects use:

```kotlin
// android/app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
        }
    }
}
```

## Secure Android Activity with excludeFromRecents

`android:excludeFromRecents="true"` removes an activity from the recents list, so its last frame isn't shown there. Most Flutter apps have one activity, so this applies to the whole app: use it when every screen is sensitive, such as banking. `FLAG_SECURE` from `screen_protector` is the narrower choice for one screen.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:excludeFromRecents="true" />
```

An app with a separate activity for sensitive flows can set it on just that one, with `android:exported="false"`.

## Biometric-Gated Encryption Key

`flutter_secure_storage` can bind stored values to a biometric or device-credential check in the OS, which is the point: a patched Dart `if` can't skip a check the Keystore enforces. Prefer this to writing a platform channel.

```dart
// lib/core/security/biometric_vault.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class BiometricVault {
  const BiometricVault();

  static const FlutterSecureStorage _storage = FlutterSecureStorage(
    aOptions: AndroidOptions.biometric(
      enforceBiometrics: true,
      requireBiometricsPerOperation: true,
      biometricPromptTitle: 'Unlock your vault',
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.passcode,
      accessControlFlags: [AccessControlFlag.biometryCurrentSet],
    ),
  );

  Future<void> save(String key, String value) =>
      _storage.write(key: key, value: value);

  Future<String?> open(String key) => _storage.read(key: key);
}
```

`AndroidOptions.biometric` uses a Keystore-based AES-GCM key. `enforceBiometrics: true` requires a secured device; `false` degrades gracefully on one without a lock screen. By default the app key is unlocked once and kept in memory, so only the first call after a start shows a prompt. `requireBiometricsPerOperation: true`, added in 11.2.0, decrypts the app key fresh for every `read`, `readAll` and `write` and never caches it, so each one prompts. It does nothing when biometrics aren't active on the device, which is one more reason to keep `enforceBiometrics: true`. `KeychainAccessibility.passcode` means the item exists only while a device passcode is set. `biometryCurrentSet` invalidates the item when the enrolled biometrics change, so a newly added fingerprint can't open it. Test on Android API 30+ and on both Face ID and Touch ID hardware.
