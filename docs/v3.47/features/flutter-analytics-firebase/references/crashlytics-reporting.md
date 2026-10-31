---
title: "Firebase Crashlytics reporting in Flutter"
author: MOZGII, Inc.
lang: en
tags: [flutter, firebase, crashlytics]
description: What to report as fatal and non-fatal, custom keys and breadcrumbs, the limits Crashlytics applies, and a small CrashService for high-risk operations.
---
# Firebase Crashlytics Reporting

## Fatal vs Non-Fatal Errors

| Scenario | Method |
|---|---|
| Unhandled Flutter framework error | `FlutterError.onError = recordFlutterFatalError`, set once in `main` |
| Unhandled Dart error outside the framework | `PlatformDispatcher.instance.onError` with `fatal: true` |
| A caught exception that affects the user | `recordError(e, stack, fatal: false)` |
| A rare state worth tracking | `recordError(e, stack, fatal: false)` |
| A business failure, such as a wrong password | Don't record it: that's expected flow |

## Crashlytics Limits

Crashlytics keeps up to 64 key-value pairs of custom keys, each up to 1 kB. Logs are capped at 64 kB per session, and older entries are dropped once a session passes that. Only the eight most recent non-fatal exceptions are stored, so recording expected failures pushes real ones out. Non-fatals wait on the device until the next fatal report goes up or the user restarts the app.

To identify a user, call `setUserIdentifier` with a pseudonymous ID, and clear it with a blank string. Clearing doesn't remove what was already recorded. Keep personal data out of keys and logs: no email, name, phone number, user content, device identifier or token.

## Custom Keys and Breadcrumbs

Set context before a high-risk step, and leave breadcrumbs through it. When a crash lands, the keys and log say what was happening.

```dart
// lib/core/crash/crash_service.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';

class CrashService {
  CrashService(this._crashlytics);

  final FirebaseCrashlytics _crashlytics;

  /// [routeTemplate] is `/orders/:id`, never a path with real IDs.
  Future<void> setContext({
    required String feature,
    String? routeTemplate,
  }) async {
    await _crashlytics.setCustomKey('feature_name', feature);
    if (routeTemplate != null) {
      await _crashlytics.setCustomKey('api_endpoint', routeTemplate);
    }
  }

  void breadcrumb(String message) => _crashlytics.log(message);

  Future<void> recordError(
    Object error,
    StackTrace? stack, {
    bool fatal = false,
    String? reason,
  }) =>
      _crashlytics.recordError(error, stack, fatal: fatal, reason: reason);
}
```

```dart
// lib/core/crash/guarded_step.dart
import 'package:yourapp/core/crash/crash_service.dart';

/// Wraps a step that may fail, recording it before passing the error on.
Future<T> guardedStep<T>(
  CrashService crash, {
  required String feature,
  required Future<T> Function() work,
}) async {
  await crash.setContext(feature: feature);
  crash.breadcrumb('$feature started');
  try {
    final result = await work();
    crash.breadcrumb('$feature finished');
    return result;
  } catch (error, stack) {
    await crash.recordError(error, stack, reason: '${feature}_failed');
    rethrow;
  }
}
```
