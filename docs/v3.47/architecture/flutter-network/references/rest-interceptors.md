---
title: Supporting Dio interceptors and adapters
author: MOZGII, Inc.
lang: en
tags: [flutter, dio, interceptors, app-check, pinning]
description: The logging and Firebase App Check interceptors for Dio, plus what NativeAdapter does and doesn't give you.
---
# Supporting Dio Interceptors and Adapters

These finish the REST client from the skill. Add them through `DioClient`'s interceptor list.

```yaml
dependencies:
  firebase_app_check: ^0.5.1
  native_dio_adapter: ^1.8.0
  pretty_dio_logger: ^1.4.0
```

## LoggingInterceptor for Debug Builds

The interceptor is silent outside `kDebugMode`, and it prints the method, the URL and the status, never a header or a body. Use `pretty_dio_logger` instead when you want bodies in debug, but keep `requestHeader: false`: the `Authorization` header holds the bearer token.

```dart
// lib/core/network/interceptors/logging_interceptor.dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';

class LoggingInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (kDebugMode) {
      debugPrint('[DIO] --> ${options.method} ${options.uri}');
    }
    handler.next(options);
  }

  @override
  void onResponse(
    Response<dynamic> response,
    ResponseInterceptorHandler handler,
  ) {
    if (kDebugMode) {
      debugPrint(
        '[DIO] <-- ${response.statusCode} ${response.requestOptions.uri}',
      );
    }
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (kDebugMode) {
      debugPrint(
        '[DIO] ERR ${err.type} ${err.requestOptions.uri}: ${err.message}',
      );
    }
    handler.next(err);
  }
}
```

## AppCheckInterceptor for Firebase App Check

The token goes on the `X-Firebase-AppCheck` header, and only for your API host, like the bearer token: a third-party URL called through this `Dio` never receives it. An App Check failure must not block the request, because the server decides what a missing token means.

```dart
// lib/core/network/interceptors/app_check_interceptor.dart
import 'package:dio/dio.dart';
import 'package:firebase_app_check/firebase_app_check.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class AppCheckInterceptor extends Interceptor {
  AppCheckInterceptor({@ignoreParam String? apiHost})
    : _apiHost =
          apiHost ?? Uri.parse(const String.fromEnvironment('API_BASE_URL')).host;

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
      // App Check failure must not block the request.
    }
    handler.next(options);
  }
}
```

## NativeAdapter and What It Doesn't Do

`NativeAdapter` from `native_dio_adapter` sends requests through the platform's stack: `cupertino_http` on iOS and macOS, `cronet_http` on Android. That brings HTTP/3 and the platform's own connection handling. `createCupertinoConfiguration` shapes the `URLSession`; the example below makes it wait for connectivity. Session settings like this pin nothing, and the adapter has no certificate callback at all. Older guides set `urlCredentialStorage` here, a setter `URLSessionConfiguration` doesn't have.

```dart
// lib/core/network/native_adapter.dart
import 'package:dio/dio.dart';
import 'package:native_dio_adapter/native_dio_adapter.dart';

/// No certificate pinning happens here: see the section below.
HttpClientAdapter buildNativeAdapter() => NativeAdapter(
  createCupertinoConfiguration: () =>
      URLSessionConfiguration.defaultSessionConfiguration()
        ..waitsForConnectivity = true,
);
```

## Where Certificate Pinning Really Happens

`IOHttpClientAdapter` is the adapter with a check, and its `validateCertificate` callback receives the leaf certificate, the host and the port. Returning `false` throws `DioException.badCertificate`. `flutter-security` has the complete setup with hashes and rotation.

dio 5.11.1 runs that callback once the response has arrived, so the request, with its bearer token, has already been sent to whoever answered. To refuse the connection during the handshake instead, give `IOHttpClientAdapter.createHttpClient` an `HttpClient` built from a `SecurityContext(withTrustedRoots: false)` that trusts only your own certificate authority.

Whichever one you use, pass the adapter to `DioClient` so the main and the plain client share it. Pinned builds can't use `NativeAdapter`.
