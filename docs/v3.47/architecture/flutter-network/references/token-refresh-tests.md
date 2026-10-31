---
title: Tests that prove token refresh and retries
author: MOZGII, Inc.
lang: en
tags: [flutter, dio, testing, auth]
description: Serverless tests with a fake Dio adapter proving one refresh for concurrent 401s, tokens kept on a network failure, only two keys deleted on rejection, and no token sent to other hosts.
---
# Tests That Prove Token Refresh and Retries

A fake `HttpClientAdapter` plays the server, so these run in `flutter test` with no network. It holds a valid token, counts refresh calls and can be told how the refresh endpoint should behave. Run them with `--dart-define=API_BASE_URL=https://api.example.com`. Imports use the package name `app`: replace it with yours.

## Concurrent 401s, Refresh Failures and Host Checks

Each test states one promise of the skill.

```dart
// test/core/network/auth_interceptor_test.dart
import 'dart:convert';
import 'dart:typed_data';

import 'package:app/core/network/dio_client.dart';
import 'package:app/core/network/interceptors/app_check_interceptor.dart';
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';

class NoAppCheck extends AppCheckInterceptor {
  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async => handler.next(options);
}

/// Plays the server. Only `Bearer fresh` is accepted on data routes.
class FakeServer implements HttpClientAdapter {
  int refreshCalls = 0;
  int refreshStatus = 200;
  bool refreshUnreachable = false;
  final seenAuth = <String, String?>{};

  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    seenAuth[options.uri.host] = options.headers['Authorization'] as String?;
    if (options.path == '/auth/refresh') {
      refreshCalls++;
      if (refreshUnreachable) {
        throw DioException.connectionError(
          requestOptions: options,
          reason: 'offline',
        );
      }
      if (refreshStatus != 200) return _json({'error': 'no'}, refreshStatus);
      await Future<void>.delayed(const Duration(milliseconds: 10));
      return _json({'access_token': 'fresh', 'refresh_token': 'rotated'}, 200);
    }
    final ok = options.headers['Authorization'] == 'Bearer fresh';
    return _json({'ok': ok}, ok ? 200 : 401);
  }

  ResponseBody _json(Map<String, Object> body, int status) =>
      ResponseBody.fromString(
        jsonEncode(body),
        status,
        headers: {
          Headers.contentTypeHeader: [Headers.jsonContentType],
        },
      );

  @override
  void close({bool force = false}) {}
}

void main() {
  late FakeServer server;
  late SessionEvents session;
  late Dio dio;
  const storage = FlutterSecureStorage();

  setUp(() {
    FlutterSecureStorage.setMockInitialValues({
      'access_token': 'expired',
      'refresh_token': 'good',
      'app_pin_hash': 'must-survive',
    });
    server = FakeServer();
    session = SessionEvents();
    dio = DioClient(storage, session, NoAppCheck(), adapter: server).instance;
  });

  test('five concurrent 401s make exactly one refresh', () async {
    final results = await Future.wait(
      List.generate(5, (i) => dio.get<dynamic>('/items/$i')),
    );
    expect(results.every((r) => r.statusCode == 200), isTrue);
    expect(server.refreshCalls, 1);
    expect(await storage.read(key: 'refresh_token'), 'rotated');
  });

  test('a network failure during refresh keeps the tokens', () async {
    server.refreshUnreachable = true;
    var expired = false;
    session.expired.listen((_) => expired = true);

    await expectLater(
      dio.get<dynamic>('/items/1'),
      throwsA(isA<DioException>()),
    );

    expect(await storage.read(key: 'refresh_token'), 'good');
    expect(expired, isFalse);
  });

  test('a 5xx from the refresh endpoint keeps the tokens', () async {
    server.refreshStatus = 503;
    await expectLater(
      dio.get<dynamic>('/items/1'),
      throwsA(isA<DioException>()),
    );
    expect(await storage.read(key: 'refresh_token'), 'good');
  });

  test('a rejected refresh removes only the two token keys', () async {
    server.refreshStatus = 401;
    var expired = false;
    session.expired.listen((_) => expired = true);

    await expectLater(
      dio.get<dynamic>('/items/1'),
      throwsA(isA<DioException>()),
    );
    await Future<void>.delayed(Duration.zero);

    expect(await storage.read(key: 'access_token'), isNull);
    expect(await storage.read(key: 'refresh_token'), isNull);
    expect(await storage.read(key: 'app_pin_hash'), 'must-survive');
    expect(expired, isTrue);
  });

  test('a third-party host never sees the token', () async {
    await expectLater(
      dio.get<dynamic>('https://other.example/x'),
      throwsA(isA<DioException>()),
    );
    expect(server.seenAuth['other.example'], isNull);
  });
}
```

## Retry Rules Without a Network

The delay is injected as zero so the test doesn't sleep. A timed-out `GET` retries until the budget runs out, and a `POST` never retries unless it carries an `Idempotency-Key`.

```dart
// test/core/network/retry_interceptor_test.dart
import 'package:app/core/network/interceptors/retry_interceptor.dart';
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';

class CapturingHandler extends ErrorInterceptorHandler {
  DioException? passed;

  @override
  void next(DioException error) => passed = error;
}

DioException timeout(RequestOptions options) => DioException(
  requestOptions: options,
  type: DioExceptionType.connectionTimeout,
);

void main() {
  late int sent;
  late RetryInterceptor retry;

  setUp(() {
    sent = 0;
    retry = RetryInterceptor(
      delay: (_) => Duration.zero,
      fetch: (options) async {
        sent++;
        return Response<dynamic>(requestOptions: options, statusCode: 200);
      },
    );
  });

  test('a GET is retried', () async {
    final options = RequestOptions(path: '/orders');
    await retry.onError(timeout(options), CapturingHandler());
    expect(sent, 1);
  });

  test('a POST is never retried', () async {
    final options = RequestOptions(path: '/orders', method: 'POST');
    final handler = CapturingHandler();
    await retry.onError(timeout(options), handler);
    expect(sent, 0);
    expect(handler.passed, isNotNull);
  });

  test('a POST with an Idempotency-Key is retried', () async {
    final options = RequestOptions(
      path: '/orders',
      method: 'POST',
      headers: {'Idempotency-Key': 'order-1'},
    );
    await retry.onError(timeout(options), CapturingHandler());
    expect(sent, 1);
  });

  test('the retry budget ends', () async {
    final options = RequestOptions(path: '/orders', extra: {'retriesLeft': 0});
    final handler = CapturingHandler();
    await retry.onError(timeout(options), handler);
    expect(sent, 0);
    expect(handler.passed, isNotNull);
  });
}
```
