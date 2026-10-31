---
title: Flutter production performance monitoring
author: MOZGII, Inc.
lang: en
tags: [flutter, performance, firebase, startup]
description: Firebase Performance traces, a Dio metric interceptor, a startup trace, a memory-pressure handler, a rate-limited frame monitor and the cold start checklist.
---
# Flutter Production Performance Monitoring

Production monitoring catches what profiling on one device misses: slow networks, old phones and flows nobody tested.

```yaml
dependencies:
  firebase_performance: ^0.12.1
  firebase_analytics: ^13.0.1
  dio: ^5.11.1
```

## Dio interceptor for Firebase HTTP metrics

`HttpMetric` records one request. This interceptor strips the query string before reporting, because a query can hold tokens or personal data, and it measures only bodies it can size. `responsePayloadSize` comes from the `Content-Length` header, which is the real count of bytes.

```dart
import 'package:dio/dio.dart';
import 'package:firebase_performance/firebase_performance.dart';

class PerformanceInterceptor extends Interceptor {
  final _metrics = <RequestOptions, HttpMetric>{};

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final metric = FirebasePerformance.instance.newHttpMetric(
      options.uri.toString().split('?').first,
      _httpMethod(options.method),
    );
    _metrics[options] = metric;
    metric.start();
    handler.next(options);
  }

  @override
  void onResponse(Response<dynamic> response, ResponseInterceptorHandler handler) {
    final metric = _metrics.remove(response.requestOptions);
    if (metric != null) {
      final length = response.headers.value(Headers.contentLengthHeader);
      metric
        ..responseContentType = response.headers.value(Headers.contentTypeHeader)
        ..httpResponseCode = response.statusCode
        ..responsePayloadSize = length == null ? null : int.tryParse(length);
      metric.stop();
    }
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final metric = _metrics.remove(err.requestOptions);
    if (metric != null) {
      metric.httpResponseCode = err.response?.statusCode;
      metric.stop();
    }
    handler.next(err);
  }

  HttpMethod _httpMethod(String method) => switch (method.toUpperCase()) {
        'POST' => HttpMethod.Post,
        'PUT' => HttpMethod.Put,
        'PATCH' => HttpMethod.Patch,
        'DELETE' => HttpMethod.Delete,
        _ => HttpMethod.Get,
      };
}
```

## Custom traces around slow work

A `Trace` measures its own duration from `start` to `stop`, so you never compute a time for it. Always stop it in `finally`. `putAttribute` takes strings for slicing the data; `setMetric` takes integers for values you want charted. Use this shape for model load time, an offline sync or image processing.

```dart
import 'package:firebase_performance/firebase_performance.dart';

Future<int> traced(String name, Future<int> Function() work) async {
  final trace = FirebasePerformance.instance.newTrace(name);
  await trace.start();
  try {
    final count = await work();
    trace.setMetric('item_count', count);
    return count;
  } finally {
    await trace.stop();
  }
}
```

## App startup trace from main to first frame

This trace measures from `main()` to the first frame the user sees. The trace's own duration is the answer. A timestamp written as an attribute is an epoch time, not a duration, so don't add one. Firebase also records an automatic `_app_start` trace, which begins earlier than Dart does.

```dart
// lib/main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import 'firebase_options.dart';

/// Yours: open whatever the first frame reads.
Future<void> openLocalStore() async {}

class StartupApp extends StatelessWidget {
  const StartupApp({super.key});

  @override
  Widget build(BuildContext context) => const MaterialApp(home: Scaffold());
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  final startup = FirebasePerformance.instance.newTrace('app_startup');
  await startup.start();

  // Do only what the first frame needs here.
  await openLocalStore();

  runApp(const ProviderScope(child: StartupApp()));
  WidgetsBinding.instance.addPostFrameCallback((_) => startup.stop());
}
```

## Memory pressure handler

`didHaveMemoryPressure` fires when the OS warns the app. Dropping the image cache is the cheapest large win. Register the observer from a keep-alive provider and read that provider once at startup, because Riverpod builds providers lazily.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

abstract interface class SyncService {
  void pauseSync();
}

/// Yours: the service and the cache to drop under memory pressure.
final syncServiceProvider = Provider<SyncService>(
  (ref) => throw UnimplementedError(),
);
final chatHistoryProvider = FutureProvider<List<String>>((ref) async => const []);

final memoryPressureProvider = Provider<MemoryPressureHandler>((ref) {
  final handler = MemoryPressureHandler(onPressure: () {
    ref.read(syncServiceProvider).pauseSync();
    ref.invalidate(chatHistoryProvider);
  });
  WidgetsBinding.instance.addObserver(handler);
  ref.onDispose(() => WidgetsBinding.instance.removeObserver(handler));
  return handler;
});

class MemoryPressureHandler with WidgetsBindingObserver {
  MemoryPressureHandler({required this.onPressure});

  final VoidCallback onPressure;

  @override
  void didHaveMemoryPressure() {
    PaintingBinding.instance.imageCache
      ..clear()
      ..clearLiveImages();
    onPressure();
  }
}
```

## Rate-limited frame budget monitor

`addTimingsCallback` delivers `FrameTiming` batches in every build mode, so it works in release. Compare `totalSpan` with the budget of the display the app runs on: a fixed 16 ms budget misses every slow frame on a 120 Hz screen. Firebase Analytics parameters take strings and numbers only, and one event per 10 seconds is plenty.

```dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:flutter/scheduler.dart';
import 'package:material_ui/material_ui.dart';

class FrameBudgetMonitor {
  static const _minEventInterval = Duration(seconds: 10);

  DateTime? _lastEvent;

  void start() => SchedulerBinding.instance.addTimingsCallback(_onTimings);

  void stop() => SchedulerBinding.instance.removeTimingsCallback(_onTimings);

  Duration get _budget {
    final views = WidgetsBinding.instance.platformDispatcher.views;
    final hz = views.isEmpty ? 60.0 : views.first.display.refreshRate;
    return Duration(microseconds: (1000000 / hz).round());
  }

  void _onTimings(List<FrameTiming> timings) {
    final budget = _budget;
    for (final timing in timings) {
      if (timing.totalSpan > budget) _report(timing);
    }
  }

  void _report(FrameTiming timing) {
    final now = DateTime.now();
    final last = _lastEvent;
    if (last != null && now.difference(last) < _minEventInterval) return;
    _lastEvent = now;
    FirebaseAnalytics.instance.logEvent(
      name: 'slow_frame',
      parameters: {
        'build_ms': timing.buildDuration.inMilliseconds,
        'raster_ms': timing.rasterDuration.inMilliseconds,
        'total_ms': timing.totalSpan.inMilliseconds,
      },
    );
  }
}
```

Start it outside debug mode only: `if (!kDebugMode) monitor.start();`.

## Cold start optimization checklist

- [ ] `main()` awaits only what the first frame needs; the rest starts after `runApp`
- [ ] The first route's widget does no heavy work in `build`
- [ ] Heavy providers on the startup path auto-dispose rather than keep alive
- [ ] No synchronous disk reads in `main()`
- [ ] Native splash set in `Info.plist` (iOS) and `styles.xml` (Android), so there's no white flash
- [ ] Any splash widget is removed as soon as hydration completes
- [ ] Measured on a release or profile build: debug start times mean nothing
