---
title: "Flutter Firebase Conventions"
author: MOZGII, Inc.
lang: en
kind: rule
description: "Firebase stack discipline for Flutter apps. Governs initialization order, config files in git, disabled-by-default collection and the Analytics consent gate, Crashlytics reporting standards, Firestore security rules checklist, and App Check enforcement. Applies to all projects using any Firebase product."
tags: ["flutter", "firebase", "crashlytics", "analytics", "firestore", "security", "gdpr"]
---
# Flutter Firebase Conventions

Applies to all Flutter projects using any Firebase product. Governs initialization order, config files, product load sequence, Crashlytics reporting standards, Analytics consent, and Firestore security rules.

---

## Initialization Order

Firebase has to be initialized before any product runs. The handlers then need to be in place before `runApp`, and consent has to apply before the first event. A Firebase call before `initializeApp` throws `No Firebase App '[DEFAULT]' has been created`.

| Step | Call | Notes |
|---|---|---|
| 1 | `Firebase.initializeApp()` | First, before any other Firebase product |
| 2 | `FirebaseAppCheck.instance.activate()` | Before the first call to a product you protect with App Check |
| 3 | `FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError` | Set before `runApp()` |
| 4 | `PlatformDispatcher.instance.onError` | Catches async errors outside the Flutter framework |
| 5 | Apply the stored consent | Before any analytics event |
| 6 | `runApp()` | After all handlers are registered |

```dart
// lib/main.dart
import 'dart:ui';

import 'package:firebase_app_check/firebase_app_check.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/core/analytics/consent_service.dart';
import 'package:yourapp/firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await Hive.openBox<Object>(ConsentService.box);

  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  await FirebaseAppCheck.instance.activate(
    providerAndroid: kDebugMode
        ? const AndroidDebugProvider()
        : const AndroidPlayIntegrityProvider(),
    providerApple: kDebugMode
        ? const AppleDebugProvider()
        : const AppleAppAttestWithDeviceCheckFallbackProvider(),
  );

  FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };

  await ConsentService.applyConsent();

  runApp(const ProviderScope(child: MaterialApp(home: Placeholder())));
}
```

---

## Firebase Config Files in Git

These files hold the project's Firebase API key and identifiers. Firebase's API key page says keys restricted to Firebase services aren't secrets and are fine in code or checked-in config files. They identify the project, and access is controlled by Security Rules, App Check and IAM. So a committed file is not a credential leak, and there's nothing to revoke.

| File | Holds | Rule |
|---|---|---|
| `google-services.json` | `current_key`, project and app IDs | Your choice per repo: fine in a private repo, or inject it in CI |
| `GoogleService-Info.plist` | `API_KEY`, project and app IDs | Same |
| `firebase_options.dart` | `apiKey`, `appId`, `messagingSenderId`, `projectId`, `storageBucket` | Commit it: the app needs it to build |

Keeping the JSON and plist out of a public repo is still reasonable hygiene, because it reduces quota abuse and hides project details. Some teams use one file per environment, injected in CI:

```bash
# Android: decode from a base64 secret
echo "$GOOGLE_SERVICES_JSON" | base64 -d > android/app/google-services.json

# iOS: decode from a base64 secret
echo "$GOOGLE_SERVICE_INFO_PLIST" | base64 -d > ios/Runner/GoogleService-Info.plist
```

Store these as `GOOGLE_SERVICES_JSON` and `GOOGLE_SERVICE_INFO_PLIST` in your CI secrets (GitHub Actions Secrets, Codemagic Environment Variables, Bitrise Secrets).

**What must stay secret:** a service account key, FCM server credentials, and the Gemini Developer API key, which Firebase says must never be in code or config files. Use separate, restricted keys for non-Firebase APIs such as Places, and keep each key's API restrictions tight. Security Rules and App Check protect your data; a hidden API key doesn't.

---

## FlutterFire CLI

Always use FlutterFire CLI to generate `firebase_options.dart`. Never write it manually.

```bash
# Install (once per machine)
dart pub global activate flutterfire_cli

# Generate from the project root
flutterfire configure
```

**Rules:**
- Run `flutterfire configure` again after adding a new Firebase product or adding a platform
- Commit `firebase_options.dart`. It contains `apiKey` along with `appId`, `messagingSenderId`, `projectId` and `storageBucket`, and that's fine: this API key isn't a secret
- Do not edit `firebase_options.dart` by hand: regenerate it

---

## Analytics Consent Gate

GDPR regions need consent before tracking data is collected. Analytics, Crashlytics and Performance Monitoring all collect by default from the first launch, before any Dart runs, so a check in Dart alone comes too late. Turn collection off in native config, then turn it on from Dart after consent. The runtime call overrides the native key and persists.

```xml
<!-- ios/Runner/Info.plist -->
<key>FIREBASE_ANALYTICS_COLLECTION_ENABLED</key>
<false/>
<key>FirebaseCrashlyticsCollectionEnabled</key>
<false/>
<key>firebase_performance_collection_enabled</key>
<false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE</key>
<false/>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside <application> -->
<meta-data android:name="firebase_analytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_crashlytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_performance_collection_enabled" android:value="false" />
<meta-data android:name="google_analytics_default_allow_analytics_storage" android:value="false" />
```

```dart
// lib/core/analytics/consent_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';

class ConsentService {
  static const box = 'consent'; // its own box: 'settings' may hold another type
  static const _analyticsKey = 'analytics_consent';

  static bool isGranted() =>
      Hive.box<Object>(box).get(_analyticsKey, defaultValue: false)! as bool;

  /// Call in main(), after Firebase.initializeApp() and before any event.
  static Future<void> applyConsent() => _setCollection(isGranted());

  static Future<void> grantConsent() async {
    await Hive.box<Object>(box).put(_analyticsKey, true);
    await _setCollection(true);
  }

  static Future<void> revokeConsent() async {
    await Hive.box<Object>(box).put(_analyticsKey, false);
    await _setCollection(false);
  }

  static Future<void> _setCollection(bool enabled) async {
    await FirebaseAnalytics.instance.setConsent(
      analyticsStorageConsentGranted: enabled,
    );
    await FirebaseAnalytics.instance.setAnalyticsCollectionEnabled(enabled);
    await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(enabled);
    await FirebasePerformance.instance.setPerformanceCollectionEnabled(enabled);
  }
}
```

**Rules by region:**

| Region | Default state | Required action |
|---|---|---|
| GDPR regions (EU, UK, EEA) | Opted-out | Require explicit opt-in before any event |
| Non-GDPR regions | May default opted-in | Must provide opt-out mechanism in settings |

**NEVER** call `analytics.logEvent()` or `analytics.logScreenView()` before `ConsentService.applyConsent()` has run. With collection off in native config, a call that slips through records nothing.

---

## Crashlytics Reporting Standards

### Fatal errors: automatic

Let Crashlytics catch fatal errors automatically via the `FlutterError.onError` and `PlatformDispatcher.instance.onError` handlers set in `main()`. Do not wrap fatal crashes.

### Non-fatal errors: explicit recording

Record a non-fatal error when a caught exception degrades the user experience.

```dart
// lib/core/analytics/crashlytics_reporter.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';

void recordNonFatal(
  Object error,
  StackTrace stack, {
  Map<String, String>? customKeys,
}) {
  if (kDebugMode) {
    debugPrint('Non-fatal: $error\n$stack');
    return;
  }
  customKeys?.forEach(FirebaseCrashlytics.instance.setCustomKey);
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: false);
}
```

**When to call `recordNonFatal`:**
- API call fails after all retries are exhausted
- Authentication failure (expired token, revoked access)
- Sync operation fails
- Any caught exception in a `try/catch` that results in a silent fallback degrading UX

**Custom keys to set BEFORE `recordError`:**

| Key | Value | Notes |
|---|---|---|
| `userId` | SHA-256 hash of user ID | Never raw PII |
| `featureName` | `"checkout"`, `"profile_sync"` | Snake-case feature name |
| `apiEndpoint` | `"/api/v2/orders"` | Path only, no query params |
| `appVersion` | `"2.4.1"` | From `packageInfo.version` |

**Limits:** Maximum 64 custom key-value pairs, each up to 1 kB. Calling `setCustomKey` again overwrites the same key.

**NEVER log in Crashlytics:**
- Email addresses, names, phone numbers
- User-generated content
- Device identifiers (IDFA, GAID, IMEI)
- Authentication tokens or session IDs

---

## Firestore Security Rules Checklist

```
[ ] rules_version = '2' declared at top of rules file
[ ] Nothing is open by default: Firestore denies a path no rule allows, so never add a broad allow
[ ] User-owned data: allow read, write: if request.auth != null && request.auth.uid == userId;
[ ] App Check enforced from the Firebase Console; rules have no request.app field
[ ] No client-side admin operations: use Cloud Functions for privileged writes
[ ] Rules tested with Firebase Emulator Suite before deploy
[ ] Rules version-controlled alongside app code (firestore.rules at repo root)
[ ] No wildcard read on entire collections without auth check
[ ] Validate incoming data shape: request.resource.data.keys().hasOnly([...])
[ ] Write-heavy paths compare request.time with a stored timestamp where a limit is needed
```

**Deploy rules:**
```bash
firebase deploy --only firestore:rules
```

**Test rules locally:**
```bash
firebase emulators:start --only firestore
```

App Check is not a rules condition. The `request` object in Firestore rules has `auth`, `method`, `path`, `query`, `resource` and `time`, so a rule that reads `request.app` is an error. Enforce App Check per product in the console.

---

## Firebase App Check

App Check helps ensure requests come from your genuine app. It isn't optional for production, and it only works once enforced.

| Platform | Production provider | Debug provider |
|---|---|---|
| Android | `AndroidPlayIntegrityProvider` | `AndroidDebugProvider` (`kDebugMode` only) |
| iOS | `AppleAppAttestWithDeviceCheckFallbackProvider` or `AppleDeviceCheckProvider` | `AppleDebugProvider` (`kDebugMode` only) |

**Rule:** The debug provider MUST be gated behind `kDebugMode`. A debug token reaches your backend without a valid device, so never ship it, never commit it to a public repo, and delete a leaked one in the console.

**Firebase Console enforcement:** Enable enforcement for each product you use, including Cloud Firestore, Cloud Storage, Realtime Database and Authentication. Cloud Functions enforce it in the function code. Read the App Check metrics first, because enforcement rejects every unverified request, including those from older app versions without App Check. It can take up to 15 minutes to apply.

The token is attached to Firebase SDK calls automatically once `activate()` completes. Your own API receives it in the `X-Firebase-AppCheck` header, verifies it with the Admin SDK, and returns 401 when it's missing or invalid.

---

## Prohibited Patterns

| Pattern | Reason |
|---|---|
| `Firebase.initializeApp()` after `runApp()` | The first Firebase call in a widget throws |
| Hardcoded Firebase project ID in source | Use `firebase_options.dart` from FlutterFire CLI |
| `analytics.logEvent()` before consent check | Collects data without consent |
| Consent handled only in Dart, with no native config | Firebase collects before the first frame |
| PII in Crashlytics logs or custom keys | Privacy regulation violation |
| Firestore access with weak rules and no App Check | Rules protect data; a hidden config file doesn't |
| A service account key or Gemini API key in the app or repo | A real secret: revoke and rotate it at once |
| Manually written `firebase_options.dart` | Maintenance hazard: use FlutterFire CLI |
| `FirebaseAppCheck.instance.activate()` after a protected product's first call | That call goes out with no token: activate in `main()` |
| `kDebugMode` check absent from debug App Check provider | Debug provider ships in production release |
