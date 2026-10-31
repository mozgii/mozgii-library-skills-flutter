---
name: flutter-network
description: Builds a production Flutter API layer - choosing REST, gRPC or GraphQL, Dio 5 with token refresh through a QueuedInterceptor, retries for idempotent methods only, gRPC with streaming auth, GraphQL with subscriptions, caching, request deduplication, an offline queue and typed response contracts. Use when setting up an API client, adding a transport, refreshing tokens on 401, adding retries, building an offline write queue, or replacing raw HTTP responses in repositories.
metadata:
  tags:
  - flutter
  - network
  - rest
  - grpc
  - graphql
  - dio
  - api
  - offline
---
# Flutter Network

The network layer stays invisible to the domain. Repositories depend on abstract interfaces, the transport is a swappable detail, and raw HTTP responses never leave `data/`. That's what lets you add gRPC to one feature without touching business logic. The hard parts are the failure paths: ten requests expiring at once, a timeout on a payment, a refresh token the server rejected. This skill gets those right.

Use it to set up an API client from scratch, add a second transport, implement token refresh with automatic retry, build an offline-first write queue, or replace raw `http.Response` returns in repositories. Skip it when the transport already works and you're only changing one endpoint's model.

## Step 0: Read Your Versions

```bash
flutter --version
flutter pub deps --style=compact | grep -E "^- (dio|grpc|protobuf|graphql_flutter|dio_cache_interceptor) "
grep -rn "Dio()" lib                      # each one bypasses your interceptors and pins
grep -rn "deleteAll()" lib                # wipes every secret the app holds
```

The pins were read from pub.dev on 2026-10-30, and the examples were compiled against them with Flutter 3.47.9 and Dart 3.13.7. The gRPC examples use real `protoc` output. Everything here was compiled, not run on a device, so try the reconnect and offline paths on a real phone.

```yaml
dependencies:
  dio: ^5.11.1
  flutter_secure_storage: ^11.2.0
  freezed_annotation: ^3.1.0
  fpdart: ^1.2.0
  injectable: ^3.0.0
  json_annotation: ^4.12.0
dev_dependencies:
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
  freezed: ^4.0.3
  injectable_generator: ^3.1.3
  json_serializable: ^6.15.2
```

## Choosing a Transport: REST vs gRPC vs GraphQL

| Criterion | REST | gRPC | GraphQL |
|---|---|---|---|
| **Use when** | Simple CRUD, human-readable payloads, team unfamiliar with protobuf | High-throughput, bidirectional streaming, microservices, low latency | Flexible queries, multiple client types, avoid over/under-fetching |
| **Payload format** | JSON | Protobuf (binary) | JSON |
| **Streaming** | SSE or polling | Native bidirectional streaming | Subscriptions via WebSocket |
| **Type safety** | Manual DTO generation | Generated from .proto | Generated from schema |
| **Tooling overhead** | Low | Medium (protoc + protoc-gen-dart) | Medium (codegen for typed operations) |
| **Flutter packages** | `dio` | `grpc` + `protobuf` | `graphql_flutter` |
| **Backend requirement** | Any HTTP server | gRPC server (Go, Rust, Node, Java) | GraphQL server (Apollo, Hasura, etc.) |
| **Firewall/proxy friendly** | Yes | Requires HTTP/2; gRPC-Web for restricted envs | Yes |

Default to REST for new projects with straightforward CRUD. Switch to gRPC when you need streaming or the backend team already has proto definitions. Switch to GraphQL when the frontend drives data shape and multiple client types exist. For a few endpoints and no interceptors, `flutter-use-http-package` is enough.

## Worked Example: REST With Dio, Token Refresh and Retries

Four files build a client that survives concurrent 401s, a dead refresh token and a flaky network.
### One Failure type for every transport

```dart
// lib/core/errors/failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failure.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network(String message) = NetworkFailure;
  const factory Failure.server(int statusCode, String message) = ServerFailure;
  const factory Failure.auth(String message) = AuthFailure;
}
```

### DioClient and the two-client rule

`DioClient` owns two `Dio` objects. The main one carries the interceptors. The plain one has none and sends the refresh and the retried request, so a 401 on either can't loop or wait on the auth queue.

Both share one `HttpClientAdapter`, so a pinned adapter covers the refresh call too. A bare `Dio()` anywhere would skip pinning.

`RetryInterceptor` needs `Dio`, and `Dio` needs its interceptors. Injecting both would make a dependency cycle that GetIt can't build, so `DioClient` constructs the interceptors itself, in its body, and hands the retry a `fetch` function. DI only sees `DioClient`.

```dart
// lib/core/network/dio_client.dart
import 'dart:async';

import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:injectable/injectable.dart';

import 'interceptors/app_check_interceptor.dart';
import 'interceptors/auth_interceptor.dart';
import 'interceptors/logging_interceptor.dart';
import 'interceptors/retry_interceptor.dart';

/// The auth layer listens to this and signs the user out.
@lazySingleton
class SessionEvents {
  final _expired = StreamController<void>.broadcast();
  Stream<void> get expired => _expired.stream;
  void expire() => _expired.add(null);
}

@module
abstract class StorageModule {
  @lazySingleton
  FlutterSecureStorage get storage => const FlutterSecureStorage();
}

@lazySingleton
class DioClient {
  DioClient(
    FlutterSecureStorage storage,
    SessionEvents session,
    AppCheckInterceptor appCheck, {
    @ignoreParam HttpClientAdapter? adapter,
  }) {
    final options = BaseOptions(
      baseUrl: const String.fromEnvironment('API_BASE_URL'),
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 15),
      headers: {'Accept': 'application/json'},
    );
    instance = Dio(options);
    if (adapter != null) instance.httpClientAdapter = adapter;
    final plain = Dio(options)..httpClientAdapter = instance.httpClientAdapter;

    instance.interceptors.addAll([
      appCheck,
      AuthInterceptor(
        storage: storage,
        plainDio: plain,
        apiHost: Uri.parse(options.baseUrl).host,
        onSessionExpired: session.expire,
      ),
      RetryInterceptor(fetch: instance.fetch),
      LoggingInterceptor(), // only active in kDebugMode
    ]);
  }

  late final Dio instance;
}
```

### Token refresh with a QueuedInterceptor

`AuthInterceptor` extends `QueuedInterceptor`. Dio runs its `onError` calls one at a time, so when ten requests fail with 401 together, exactly one refreshes. The others find a newer token in storage and just retry.

The important branch is what happens when the refresh itself fails, because two very different things look alike:

- **The server rejected the refresh token** (400, 401 or 403). The session is over, so delete those two keys and emit `expire`.
- **The refresh didn't reach a verdict** (timeout, offline, 5xx). The session may be fine, so keep the tokens and pass the error up.

Never call `deleteAll()` here. It wipes every key the app holds, including a PIN hash and a device token, and a Wi-Fi drop mustn't sign anyone out. The token is attached only to your API host, so a request to a third-party URL never carries it.

```dart
// lib/core/network/interceptors/auth_interceptor.dart
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor({
    required FlutterSecureStorage storage,
    required Dio plainDio,
    required String apiHost,
    required void Function() onSessionExpired,
  }) : _storage = storage,
       _plainDio = plainDio,
       _apiHost = apiHost,
       _onSessionExpired = onSessionExpired;

  final FlutterSecureStorage _storage;
  final Dio _plainDio;
  final String _apiHost;
  final void Function() _onSessionExpired;

  static const _tokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _rejected = {400, 401, 403};

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _storage.read(key: _tokenKey);
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
    final request = err.requestOptions;
    if (err.response?.statusCode != 401 || request.uri.host != _apiHost) {
      return handler.next(err);
    }

    try {
      final stored = await _storage.read(key: _tokenKey);
      final refreshedMeanwhile =
          stored != null &&
          request.headers['Authorization'] != 'Bearer $stored';
      final token = refreshedMeanwhile ? stored : await _refresh();
      if (token == null) {
        await _endSession();
        return handler.next(err);
      }

      request.headers['Authorization'] = 'Bearer $token';
      if (request.data is FormData) {
        request.data = (request.data as FormData).clone();
      }
      return handler.resolve(await _plainDio.fetch<dynamic>(request));
    } on DioException catch (e) {
      if (e.requestOptions.path == '/auth/refresh' &&
          _rejected.contains(e.response?.statusCode)) {
        await _endSession();
        return handler.next(err);
      }
      return handler.next(e);
    }
  }

  /// Returns the new access token, or null when there is no refresh token.
  Future<String?> _refresh() async {
    final refreshToken = await _storage.read(key: _refreshTokenKey);
    if (refreshToken == null) return null;

    final response = await _plainDio.post<Map<String, dynamic>>(
      '/auth/refresh',
      data: {'refresh_token': refreshToken},
    );
    final body = response.data!;
    final access = body['access_token'] as String;
    await _storage.write(key: _tokenKey, value: access);
    // A server that rotates refresh tokens sends a new one each time.
    final rotated = body['refresh_token'] as String?;
    if (rotated != null) {
      await _storage.write(key: _refreshTokenKey, value: rotated);
    }
    return access;
  }

  Future<void> _endSession() async {
    await _storage.delete(key: _tokenKey);
    await _storage.delete(key: _refreshTokenKey);
    _onSessionExpired();
  }
}
```

### Retries for idempotent methods only

A retry repeats a request the server may already have processed. `GET`, `HEAD`, `PUT`, `DELETE` and `OPTIONS` are idempotent, so repeating them is safe. Re-sending a `POST` after a timeout can create a second order, so a `POST` is retried only when it carries an `Idempotency-Key` that your server honours.

Backoff is exponential with jitter: `2^n * 1000 ms` plus up to 500 ms, so a thousand phones don't return in step. It retries timeouts, connection errors and 502, 503 and 504.

```dart
// lib/core/network/interceptors/retry_interceptor.dart
import 'dart:math';

import 'package:dio/dio.dart';

typedef Fetch = Future<Response<dynamic>> Function(RequestOptions options);

class RetryInterceptor extends Interceptor {
  RetryInterceptor({
    required Fetch fetch,
    Duration Function(int attempt)? delay,
  }) : _fetch = fetch,
       _delay = delay ?? _backoff;

  final Fetch _fetch;
  final Duration Function(int attempt) _delay;

  static const _maxRetries = 3;
  static const _idempotent = {'GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS'};
  static const _retryableStatus = {502, 503, 504};
  static final _random = Random();

  static Duration _backoff(int attempt) => Duration(
    milliseconds: (pow(2, attempt) * 1000).toInt() + _random.nextInt(500),
  );

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
    final extra = err.requestOptions.extra;
    final retriesLeft = extra['retriesLeft'] as int? ?? _maxRetries;
    if (!_isRetryable(err) || retriesLeft == 0) return handler.next(err);

    await Future<void>.delayed(_delay(_maxRetries - retriesLeft + 1));
    extra['retriesLeft'] = retriesLeft - 1;
    try {
      return handler.resolve(await _fetch(err.requestOptions));
    } on DioException catch (e) {
      return handler.next(e);
    }
  }
}
```

The logging and App Check interceptors are in `references/rest-interceptors.md`.

## Certificate Pinning With Dio

A pin only exists where code checks one. A `NativeAdapter` has no certificate callback, so a `URLSessionConfiguration` passed to it pins nothing, whatever a comment beside it claims.

Real pinning uses `IOHttpClientAdapter.validateCertificate`, set up in `flutter-security`. Assign that adapter through the `adapter` parameter of `DioClient`, and the plain client picks it up too. dio runs `validateCertificate` after the response arrives, so the request has already gone out when the check fails. See `references/rest-interceptors.md`.

## Other Transports and Layers

Each fits the same repository contract.

| Topic | Reference |
|---|---|
| gRPC setup, streaming auth, error mapping, bidirectional streaming, gRPC-Web | `references/grpc.md` |
| GraphQL client, subscriptions with auth, error mapping | `references/graphql.md` |
| Response caching and request deduplication | `references/caching-and-deduplication.md` |
| Offline write queue | `references/offline-queue.md` |
| `ApiResponse`, the error envelope, repository rules | `references/api-contracts.md` |

## Verify

```bash
dart run build_runner build
flutter analyze
flutter test
grep -rn "deleteAll()" lib      # expect nothing
grep -rn "Dio()" lib            # expect nothing outside tests
```

`references/token-refresh-tests.md` holds the tests to copy. They need no server, and they prove the claims above: five concurrent 401s make one refresh, a network failure keeps the tokens, a rejected refresh removes only the two token keys, and a third-party host never sees the token. Then force a 401 on a real build by editing the stored token.

## Gotchas

1. **Everyone is signed out when the Wi-Fi drops.** Symptom: logouts cluster around network blips. Cause: any refresh failure ran the sign-out path. Fix: end the session only on 400, 401 or 403 from the refresh endpoint.
2. **The refresh endpoint is called five times at once.** Symptom: a rotating server invalidates the token and users are thrown out. Cause: a plain `Interceptor` handles 401s in parallel. Fix: extend `QueuedInterceptor`.
3. **Requests hang forever after a 401.** Symptom: a spinner that never ends and no error. Cause: the refresh went through the same `Dio`, whose 401 waits on the queue that's waiting for it. Fix: use the plain client.
4. **A customer has two identical orders.** Symptom: duplicates after a slow network. Cause: the retry re-sent a `POST`. Fix: retry idempotent methods only, or send an `Idempotency-Key`.
5. **The app crashes at startup while GetIt resolves `Dio`.** Symptom: a stack overflow, or a dependency cycle reported in the injectable output. Cause: `Dio` needs the interceptor and the interceptor needs `Dio`. Fix: construct the interceptors inside `DioClient`.
6. **`The MultipartFile has already been finalized`.** Symptom: an upload fails only after a token refresh. Cause: a `FormData` stream can't be read twice. Fix: send `FormData.clone()` on the retry.
7. **`Functions marked 'async' must have a return type which is a supertype of 'Future'`.** Symptom: `interceptUnary` won't compile with `async`. Cause: it must return a `ResponseFuture`, and `async` gives a plain `Future`. Fix: add the token in a `CallOptions` provider.
8. **A subscription connects and then gets no data.** Symptom: queries work and subscriptions are refused. Cause: `AuthLink` headers don't reach the socket. Fix: send the token in `initialPayload`.

## Anti-Patterns

| Don't | Do instead |
|---|---|
| `deleteAll()` when a refresh fails | Delete the two token keys, only on a real rejection |
| A bare `Dio()` for a refresh or retry | The plain client that shares the adapter |
| Retry every failed request | Idempotent methods, or an `Idempotency-Key` |
| Print request headers in logs | Log the method, path and status only |
| Return `Response` or `Map` from a repository | `Future<Either<Failure, T>>` with an entity |
| Cache a 401 or 403 response | `hitCacheOnErrorCodes` lists 5xx only |
| Attach a token to every host | Compare `uri.host` with your API host |

## Pre-Merge Checklist

- [ ] The tests in `references/token-refresh-tests.md` pass
- [ ] Refresh goes through a client with no auth interceptor
- [ ] A refresh that merely failed keeps the tokens
- [ ] Only idempotent requests retry
- [ ] gRPC streaming calls carry auth, not only unary ones
- [ ] Pinned builds use the same adapter on both clients
- [ ] No token, password or body reaches a release log
- [ ] The repository returns entities and `Failure`, never transport types

## See Also

`flutter-security` (certificate pinning with `validateCertificate`), `flutter-auth` (sign-in and session state), `flutter-core` (architecture), `flutter-database` (local persistence for offline-first data), `flutter-use-http-package` (the smaller `http` client).

Depth in this skill: `references/rest-interceptors.md`, `references/token-refresh-tests.md`, `references/grpc.md`, `references/graphql.md`, `references/caching-and-deduplication.md`, `references/offline-queue.md`, `references/api-contracts.md`.
