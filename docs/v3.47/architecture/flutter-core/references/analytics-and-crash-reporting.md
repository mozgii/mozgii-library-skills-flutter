---
title: Analytics and crash reporting wiring
author: MOZGII, Inc.
lang: en
tags: [flutter, firebase, analytics, crashlytics]
description: Screen tracking through go_router, a typed analytics service whose parameters pass Firebase's type check, and global error capture with Crashlytics.
---
# Analytics and Crash Reporting Wiring

Analytics and crash reporting belong in `core/` so features call one typed service and never touch Firebase directly. `flutter-analytics-firebase` goes further on event design, and it has the consent gate this wiring needs: collection stays off in native config until the person agrees, because all three Firebase products start before your first Dart frame.

## Firebase Analytics Screen Tracking With go_router

A `NavigatorObserver` sees every route change and logs its name. `NavigatorObserver` lives in the widgets library, so this file needs no Material import.

```dart
// lib/core/analytics/go_router_observer.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:flutter/widgets.dart';

class GoRouterObserver extends NavigatorObserver {
  GoRouterObserver({FirebaseAnalytics? analytics})
    : _analytics = analytics ?? FirebaseAnalytics.instance;

  final FirebaseAnalytics _analytics;

  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    _logScreen(route);
  }

  @override
  void didReplace({Route<dynamic>? newRoute, Route<dynamic>? oldRoute}) {
    if (newRoute != null) _logScreen(newRoute);
  }

  void _logScreen(Route<dynamic> route) {
    final name = route.settings.name;
    if (name != null && name.isNotEmpty) {
      _analytics.logScreenView(screenName: name);
    }
  }
}
```

Wire it in with `GoRouter(observers: [GoRouterObserver()], routes: $appRoutes)`. Since `go_router` 17, `ShellRoute` navigation reaches these observers too.

## Crashlytics Global Error Handling

Two hooks catch everything. `FlutterError.onError` takes framework errors such as widget build and render failures, and `PlatformDispatcher.instance.onError` takes unhandled async errors from a `Future`, a `Stream` or an isolate. Crashlytics records in release only. Returning `true` suppresses the default error handler.

```dart
// lib/core/errors/error_handler.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';

void setupErrorHandling() {
  if (kDebugMode) return; // Crashlytics only in release

  // Flutter framework errors (widget build failures, render errors)
  FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

  // Unhandled async errors (Future, Stream, isolate errors)
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };
}
```

Call it once in `main()` before `runApp()`, after `Firebase.initializeApp`:

```dart
// lib/main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import 'app.dart';
import 'core/di/injection.dart';
import 'core/errors/error_handler.dart';
import 'firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  setupErrorHandling();
  await configureDependencies();
  runApp(const ProviderScope(child: MyApp()));
}
```

## AnalyticsService: A Typed Event Class With String or num Parameters

`logEvent` accepts `Map<String, Object>`, but Firebase Analytics asserts that every value is a `String` or a `num`. A `bool` compiles, then fails that assert in debug with `'string' OR 'number' must be set as the value of the parameter`. Send flags as `1` or `0`, which you can also sum in reports. The same check covers `AnalyticsEventItem.parameters`.

```dart
// lib/core/analytics/analytics_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class AnalyticsService {
  AnalyticsService({@ignoreParam FirebaseAnalytics? analytics})
    : _analytics = analytics ?? FirebaseAnalytics.instance;

  final FirebaseAnalytics _analytics;

  Future<void> logScreenView(String screenName) =>
      _analytics.logScreenView(screenName: screenName);

  Future<void> logAuthEvent({required String method, required bool success}) =>
      _analytics.logEvent(
        name: 'auth_attempt',
        parameters: {'method': method, 'success': success ? 1 : 0},
      );

  Future<void> logAiEvent({
    required String eventName,
    required Map<String, Object> parameters,
  }) => _analytics.logEvent(name: eventName, parameters: parameters);

  Future<void> logPurchaseEvent({
    required String productId,
    required double price,
    required String currency,
  }) => _analytics.logPurchase(
    currency: currency,
    value: price,
    items: [AnalyticsEventItem(itemId: productId)],
  );

  Future<void> logError({required String errorType}) => _analytics.logEvent(
    name: 'app_error',
    parameters: {'error_type': errorType},
  );
}
```

Keep names, emails and message bodies out of every parameter: analytics isn't a place for personal data. So `logError` takes a short error type and no free text, because an error message can quote what the person typed.
