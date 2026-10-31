---
title: The API client stack in core/network
author: MOZGII, Inc.
lang: en
tags: [flutter, dio, grpc, interceptors, di]
description: How DioClient stacks its interceptors, a retry that only touches idempotent methods, token refresh with a QueuedInterceptor, and the ApiClient facade over REST and gRPC.
---
# The API Client Stack in core/network

This is the core wiring: one `Dio`, a fixed interceptor order, and a facade that hides the transport. `flutter-network` covers transports in depth; everything here works without it.

```
core/network/
  dio_client.dart       # base Dio with all interceptors stacked
  grpc_client.dart      # base gRPC channel with auth metadata
  api_client.dart       # facade routing to Dio or gRPC based on config
  interceptors/
    auth_interceptor.dart
    retry_interceptor.dart
    logging_interceptor.dart
    app_check_interceptor.dart
    performance_interceptor.dart
```

```yaml
dependencies:
  firebase_analytics: ^13.0.1
  firebase_crashlytics: ^6.0.1
  firebase_app_check: ^0.5.1
  firebase_performance: ^0.12.1
  dio: ^5.11.1
  grpc: ^5.2.0
  pretty_dio_logger: ^1.4.0
```

## ApiConfig and the Module That Registers It

`ApiConfig` holds what changes per environment. The injectable module registers it under a name, along with the gRPC channel and the `Dio` that repositories receive.

```dart
// lib/core/network/api_config.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_config.freezed.dart';

@freezed
abstract class ApiConfig with _$ApiConfig {
  const factory ApiConfig({
    required String baseUrl,
    @Default(false) bool useGrpc,
    @Default(30) int connectTimeoutSeconds,
    @Default(60) int receiveTimeoutSeconds,
  }) = _ApiConfig;
}
```

```dart
// lib/core/network/network_module.dart
import 'package:dio/dio.dart';
import 'package:grpc/grpc.dart';
import 'package:injectable/injectable.dart';

import 'api_config.dart';
import 'dio_client.dart';

@module
abstract class NetworkModule {
  @Named('apiConfig')
  @lazySingleton
  ApiConfig get apiConfig =>
      const ApiConfig(baseUrl: String.fromEnvironment('API_BASE_URL'));

  @lazySingleton
  ClientChannel get grpcChannel => ClientChannel(
    const String.fromEnvironment('GRPC_HOST'),
    options: const ChannelOptions(credentials: ChannelCredentials.secure()),
  );

  @lazySingleton
  Dio dio(DioClient client) => client.instance;
}
```

## DioClient and the Interceptor Order

`DioClient` stacks five interceptors in a fixed order: the bearer token, the App Check token, the retry on network errors and 5xx, Firebase Performance metrics, then readable logs in debug only. The retry needs to re-send through this same `Dio`, and an interceptor that asks DI for `Dio` while `Dio` asks DI for the interceptor is a cycle. So `DioClient` creates the auth and retry interceptors itself and hands the retry `fetch`, and DI never sees either one.

```dart
// lib/core/network/dio_client.dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';
import 'package:injectable/injectable.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';

import 'api_config.dart';
import 'interceptors/app_check_interceptor.dart';
import 'interceptors/auth_interceptor.dart';
import 'interceptors/performance_interceptor.dart';
import 'interceptors/retry_interceptor.dart';

@lazySingleton
class DioClient {
  DioClient(
    TokenStorage tokenStorage,
    PerformanceInterceptor performanceInterceptor,
    @Named('apiConfig') ApiConfig config,
  ) {
    final options = BaseOptions(
      baseUrl: config.baseUrl,
      connectTimeout: Duration(seconds: config.connectTimeoutSeconds),
      receiveTimeout: Duration(seconds: config.receiveTimeoutSeconds),
      headers: {'Accept': 'application/json'},
    );
    instance = Dio(options);
    final apiHost = Uri.parse(options.baseUrl).host;
    // Sends retries past the auth queue, on the same adapter as the main client.
    final plain = Dio(options)..httpClientAdapter = instance.httpClientAdapter;

    instance.interceptors.addAll([
      // 1. Attach Bearer token to your API only, refresh on 401
      AuthInterceptor(tokenStorage, plain, apiHost),
      AppCheckInterceptor(apiHost), // 2. App Check token, your API only
      RetryInterceptor(fetch: instance.fetch), // 3. Retry idempotent failures
      performanceInterceptor, // 4. Firebase Performance HTTP metrics
      if (kDebugMode)
        PrettyDioLogger(
          // 5. Human-readable logs in debug only
          requestHeader: false, // would print the Bearer token
          requestBody: true,
          responseBody: true,
        ),
    ]);
  }

  late final Dio instance;
}
```

## Token Refresh With a QueuedInterceptor

`AuthInterceptor` extends `QueuedInterceptor`, so ten requests that fail with 401 at once are handled one at a time: the first refreshes, and the rest see a newer token and just retry. It sends the retry through a plain `Dio`, which has no interceptors, because a request through the same `Dio` could wait on this queue forever. The plain client shares the main client's adapter, since a bare `Dio()` would skip any certificate pinning. The token goes only to your API host, so a third-party URL called through this `Dio` never receives it. Bind your own class to `TokenStorage` with `@LazySingleton(as: TokenStorage)`; until you do, injectable warns about a missing dependency.

```dart
// lib/core/network/interceptors/auth_interceptor.dart
import 'package:dio/dio.dart';

abstract interface class TokenStorage {
  String? get accessToken;
  Future<String> refresh();
}

class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor(this._tokenStorage, this._plainDio, this._apiHost);

  final TokenStorage _tokenStorage;
  final Dio _plainDio;
  final String _apiHost;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = _tokenStorage.accessToken;
    if (token != null && options.uri.host == _apiHost) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    if (err.response?.statusCode != 401 ||
        err.requestOptions.uri.host != _apiHost) {
      return handler.next(err);
    }

    final sent = err.requestOptions.headers['Authorization'];
    try {
      var token = _tokenStorage.accessToken;
      if (token == null || sent == 'Bearer $token') {
        token = await _tokenStorage.refresh();
      }
      err.requestOptions.headers['Authorization'] = 'Bearer $token';
      handler.resolve(await _plainDio.fetch<dynamic>(err.requestOptions));
    } on DioException catch (e) {
      // A failed refresh propagates so the auth layer can decide to sign out.
      handler.next(e);
    }
  }
}
```

## App Check Interceptor

The App Check token goes only to your API host, like the bearer token, so a third-party URL called through this `Dio` never receives it. A failure here must not block the request: the server decides what to do with a missing token.

```dart
// lib/core/network/interceptors/app_check_interceptor.dart
import 'package:dio/dio.dart';
import 'package:firebase_app_check/firebase_app_check.dart';

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
      if (token != null) {
        options.headers['X-Firebase-AppCheck'] = token;
      }
    } catch (_) {
      // App Check failure should not block the request.
    }
    handler.next(options);
  }
}
```

## Retry Interceptor for Idempotent Methods Only

A retry repeats a request the server may already have processed. `GET`, `HEAD`, `PUT`, `DELETE` and `OPTIONS` are idempotent, so repeating them is safe. `POST` isn't: re-sending one after a timeout can charge a card twice. A `POST` is retried only if it carries an `Idempotency-Key` header that the server honours.

The interceptor retries timeouts, connection errors and 502, 503 and 504, with a growing delay, and takes `fetch` as a function, which also makes it testable without a network.

```dart
// lib/core/network/interceptors/retry_interceptor.dart
import 'package:dio/dio.dart';

typedef Fetch = Future<Response<dynamic>> Function(RequestOptions options);

class RetryInterceptor extends Interceptor {
  RetryInterceptor({required Fetch fetch}) : _fetch = fetch;

  final Fetch _fetch;
  static const int _maxRetries = 3;
  static const _idempotent = {'GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS'};
  static const _retryableStatus = {502, 503, 504};

  bool _isRetryable(DioException err) {
    final request = err.requestOptions;
    final safe =
        _idempotent.contains(request.method.toUpperCase()) ||
        request.headers.containsKey('Idempotency-Key');
    if (!safe) return false;
    return err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout ||
        err.type == DioExceptionType.connectionError ||
        _retryableStatus.contains(err.response?.statusCode);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    final attempt = (err.requestOptions.extra['retry_attempt'] as int?) ?? 0;
    if (!_isRetryable(err) || attempt >= _maxRetries) return handler.next(err);

    await Future<void>.delayed(Duration(milliseconds: 500 * (attempt + 1)));
    err.requestOptions.extra['retry_attempt'] = attempt + 1;
    try {
      handler.resolve(await _fetch(err.requestOptions));
    } on DioException catch (e) {
      handler.next(e);
    }
  }
}
```

The same behaviour, proved without a network, is in `references/testing-setup.md`.

## Firebase Performance HTTP Metrics

`PerformanceInterceptor` records one `HttpMetric` per request and keeps it in `extra` between the request and the response. The URL goes in without its query string, which can hold tokens or personal data.

```dart
// lib/core/network/interceptors/performance_interceptor.dart
import 'package:dio/dio.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class PerformanceInterceptor extends Interceptor {
  static const _key = 'perf_metric';

  HttpMethod _method(String method) => HttpMethod.values.firstWhere(
    (m) => m.name.toUpperCase() == method.toUpperCase(),
    orElse: () => HttpMethod.Get,
  );

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final metric = FirebasePerformance.instance.newHttpMetric(
      options.uri.toString().split('?').first,
      _method(options.method),
    );
    options.extra[_key] = metric;
    await metric.start();
    handler.next(options);
  }

  @override
  Future<void> onResponse(
    Response<dynamic> response,
    ResponseInterceptorHandler handler,
  ) async {
    await _stop(response.requestOptions, response.statusCode);
    handler.next(response);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    await _stop(err.requestOptions, err.response?.statusCode);
    handler.next(err);
  }

  Future<void> _stop(RequestOptions options, int? status) async {
    final metric = options.extra.remove(_key);
    if (metric is HttpMetric) {
      metric.httpResponseCode = status;
      await metric.stop();
    }
  }
}
```

## ApiClient Facade Over REST and gRPC

Repositories take `ApiClient` when a feature may move between transports. `rest` always uses Dio, and `grpc` is available when `useGrpc` is true in `ApiConfig`.

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';
import 'package:grpc/grpc.dart';
import 'package:injectable/injectable.dart';

import 'api_config.dart';
import 'dio_client.dart';

@lazySingleton
class ApiClient {
  ApiClient(this._dio, this._grpcChannel, @Named('apiConfig') ApiConfig config)
    : _useGrpc = config.useGrpc;

  final DioClient _dio;
  final ClientChannel _grpcChannel;
  final bool _useGrpc;

  /// REST endpoint: always uses Dio.
  Dio get rest => _dio.instance;

  /// gRPC channel: available when useGrpc is true in ApiConfig.
  ClientChannel get grpc {
    assert(_useGrpc, 'gRPC is disabled in ApiConfig');
    return _grpcChannel;
  }
}
```

A `RetryInterceptor` request built with `FormData` can't be sent twice, since its stream is already consumed. Clone it in the retry path with `FormData.clone()` if your app retries uploads.
