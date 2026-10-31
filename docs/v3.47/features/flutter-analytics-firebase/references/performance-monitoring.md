---
title: "Firebase Performance Monitoring in Flutter"
author: MOZGII, Inc.
lang: en
tags: [flutter, firebase, performance, dio]
description: HTTP metrics through a Dio interceptor with the right HttpMetric setters and no query strings, custom traces, and measuring startup without calling Firebase too early.
---
# Firebase Performance Monitoring

Performance Monitoring records an automatic `_app_start` trace, from the app opening until it's responsive, plus any traces you add. `setPerformanceCollectionEnabled` turns collection on or off, and a change takes effect after a restart.

## HTTP Traces with a Dio Interceptor

The interceptor keeps each metric in `RequestOptions.extra`, which travels with the request, so nothing depends on an object's `hashCode`. The URL loses its query string, because tokens and search terms often live there. `HttpMetric`'s setter is `httpResponseCode`, and it takes a nullable int: leave it null on a network failure rather than inventing a code.

```dart
// lib/core/network/performance_interceptor.dart
import 'package:dio/dio.dart';
import 'package:firebase_performance/firebase_performance.dart';

class PerformanceInterceptor extends Interceptor {
  static const _key = 'perf_metric';

  HttpMethod _methodOf(RequestOptions options) => HttpMethod.values.firstWhere(
        (method) => method.name.toUpperCase() == options.method.toUpperCase(),
        orElse: () => HttpMethod.Get,
      );

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final url = options.uri.replace(query: '').toString();
    final metric = FirebasePerformance.instance.newHttpMetric(
      url,
      _methodOf(options),
    );
    await metric.start();
    options.extra[_key] = metric;
    handler.next(options);
  }

  @override
  Future<void> onResponse(
    Response<dynamic> response,
    ResponseInterceptorHandler handler,
  ) async {
    final metric = response.requestOptions.extra.remove(_key) as HttpMetric?;
    if (metric != null) {
      metric
        ..httpResponseCode = response.statusCode
        ..responseContentType = response.headers.value('content-type')
        ..responsePayloadSize = _contentLength(response);
      await metric.stop();
    }
    handler.next(response);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    final metric = err.requestOptions.extra.remove(_key) as HttpMetric?;
    if (metric != null) {
      metric.httpResponseCode = err.response?.statusCode;
      await metric.stop();
    }
    handler.next(err);
  }

  int? _contentLength(Response<dynamic> response) {
    final header = response.headers.value('content-length');
    return header == null ? null : int.tryParse(header);
  }
}
```

## Custom Traces

Wrap a slow operation in a named trace. The `finally` stops the trace even when the work throws. Names follow the same no-personal-data rule as events.

```dart
// lib/core/analytics/performance_service.dart
import 'package:firebase_performance/firebase_performance.dart';

class PerformanceService {
  PerformanceService(this._performance);

  final FirebasePerformance _performance;

  Future<T> trace<T>(String name, Future<T> Function() work) async {
    final trace = _performance.newTrace(name);
    await trace.start();
    try {
      return await work();
    } finally {
      await trace.stop();
    }
  }
}
```

## App Startup Trace

The automatic `_app_start` trace is usually enough. To measure your own Dart startup as well, start after `Firebase.initializeApp`: a trace created before it throws `No Firebase App '[DEFAULT]' has been created`. Use a `Stopwatch` for the time before that point and attach it as a metric.

```dart
// lib/main_trace.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/firebase_options.dart';

Future<void> main() async {
  final beforeFirebase = Stopwatch()..start();
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  final trace = FirebasePerformance.instance.newTrace('dart_startup');
  await trace.start();
  trace.setMetric('before_firebase_ms', beforeFirebase.elapsedMilliseconds);

  runApp(const MaterialApp(home: Placeholder()));

  WidgetsBinding.instance.addPostFrameCallback((_) => trace.stop());
}
```
