---
title: Testing setup for the core layers
author: MOZGII, Inc.
lang: en
tags: [flutter, testing, mocktail]
description: How test folders mirror lib, a use case test with mocktail, a retry test that proves a POST is never re-sent, and attack tests that keep the bearer token on your API host.
---
# Testing Setup for the Core Layers

The test tree mirrors `lib/`, so the test for a file is always easy to find.

```
test/
  features/
    auth/
      data/
        auth_repository_impl_test.dart
      domain/
        sign_in_usecase_test.dart
      presentation/
        auth_notifier_test.dart
  core/
    router/
      router_test.dart
```

```yaml
dev_dependencies:
  mocktail: ^1.0.5
```

## Use Case Unit Test With mocktail

A use case test proves the contract: success carries the entity, and failure arrives as a `Left` rather than an exception. Mock the repository interface, never the implementation.

```dart
// test/features/auth/domain/sign_in_usecase_test.dart
import 'package:app/core/errors/failure.dart';
import 'package:app/features/auth/domain/entities/user.dart';
import 'package:app/features/auth/domain/repositories/auth_repository.dart';
import 'package:app/features/auth/domain/usecases/sign_in_usecase.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fpdart/fpdart.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late MockAuthRepository mockRepository;
  late SignInUseCase useCase;
  const tUser = User(id: '1', email: 'test@example.com', displayName: 'Test');
  const params = SignInParams(email: 'test@example.com', password: '123');

  setUp(() {
    mockRepository = MockAuthRepository();
    useCase = SignInUseCase(mockRepository);
  });

  group('SignInUseCase', () {
    test('returns User on success', () async {
      when(() => mockRepository.signIn(any(), any()))
          .thenAnswer((_) async => right(tUser));

      final result = await useCase(params);

      expect(result, right<Failure, User>(tUser));
    });

    test('returns NetworkFailure on network error', () async {
      when(
        () => mockRepository.signIn(any(), any()),
      ).thenAnswer((_) async => left(const Failure.network('No connection')));

      final result = await useCase(params);

      expect(result.isLeft(), true);
    });
  });
}
```

## Router Guard Test: One GoRouter Across Sign-In

This test proves the guard in `routing-and-login-screen.md` on a real widget tree. It overrides the use case so no network or GetIt is involved, then checks three things: a signed-out user lands on the login route, signing in reaches home, and the `GoRouter` object is the same one throughout. A second test covers the guard's path check, a pure function: `/authors` is the case a bare prefix check gets wrong.

```dart
// test/core/router/router_test.dart
import 'package:app/core/errors/failure.dart';
import 'package:app/core/router/router.dart';
import 'package:app/features/auth/domain/entities/user.dart';
import 'package:app/features/auth/domain/repositories/auth_repository.dart';
import 'package:app/features/auth/domain/usecases/sign_in_usecase.dart';
import 'package:app/features/auth/presentation/providers/auth_provider.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fpdart/fpdart.dart';
import 'package:material_ui/material_ui.dart';

class FakeAuthRepository implements AuthRepository {
  @override
  Future<Either<Failure, User>> signIn(String email, String password) async =>
      right(const User(id: '1', email: 'a@example.com', displayName: 'A'));
}

void main() {
  test('only /auth and what sits under it count as auth routes', () {
    expect(isAuthLocation('/auth'), isTrue);
    expect(isAuthLocation('/auth/login'), isTrue);
    expect(isAuthLocation('/authors'), isFalse);
    expect(isAuthLocation('/'), isFalse);
  });

  testWidgets('auth changes redirect without rebuilding the router', (
    tester,
  ) async {
    final container = ProviderContainer(
      overrides: [
        signInUseCaseProvider.overrideWithValue(
          SignInUseCase(FakeAuthRepository()),
        ),
      ],
    );
    addTearDown(container.dispose);

    final router = container.read(routerProvider);
    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: MaterialApp.router(routerConfig: router),
      ),
    );
    await tester.pumpAndSettle();
    expect(router.state.matchedLocation, '/auth/login');

    await container.read(authProvider.notifier).signIn('a@example.com', 'pw');
    await tester.pumpAndSettle();

    expect(router.state.matchedLocation, '/');
    expect(identical(container.read(routerProvider), router), isTrue);
  });
}
```

## Retry Test: A POST Is Never Re-Sent

This test drives `RetryInterceptor` with a counting `fetch` and no network. A timed-out `GET` is sent again, and a `POST` that fails the same way is passed through at once. A `POST` carrying an `Idempotency-Key` is the one exception. Test imports use the package name `app`: replace it with yours.

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

DioException failure(String method, {Map<String, dynamic>? headers}) {
  final options = RequestOptions(
    path: '/orders',
    method: method,
    headers: headers,
  );
  return DioException(
    requestOptions: options,
    type: DioExceptionType.connectionTimeout,
  );
}

void main() {
  test('POST without an Idempotency-Key is never re-sent', () async {
    var sent = 0;
    final retry = RetryInterceptor(
      fetch: (options) async {
        sent++;
        return Response<dynamic>(requestOptions: options, statusCode: 200);
      },
    );

    final handler = CapturingHandler();
    await retry.onError(failure('POST'), handler);

    expect(sent, 0);
    expect(handler.passed, isNotNull);
  });

  test('GET is re-sent', () async {
    var sent = 0;
    final retry = RetryInterceptor(
      fetch: (options) async {
        sent++;
        return Response<dynamic>(requestOptions: options, statusCode: 200);
      },
    );

    await retry.onError(failure('GET'), CapturingHandler());

    expect(sent, 1);
  });

  test('POST with an Idempotency-Key is re-sent', () async {
    var sent = 0;
    final retry = RetryInterceptor(
      fetch: (options) async {
        sent++;
        return Response<dynamic>(requestOptions: options, statusCode: 200);
      },
    );

    await retry.onError(
      failure('POST', headers: {'Idempotency-Key': 'order-1'}),
      CapturingHandler(),
    );

    expect(sent, 1);
  });
}
```

## Token Host Attack Test: A Third-Party Host Never Gets the Bearer Token

`AuthInterceptor` attaches the token and refreshes it only for your API host, so these tests attack that check. Each URL below is a host that isn't yours, and two of them are built to look like it. A 401 from such a host has to pass straight through, because a hostile server could otherwise trigger a refresh and receive the new token on the retry. The last test is the control: the same 401 from your own API does refresh.

```dart
// test/core/network/auth_interceptor_test.dart
import 'dart:typed_data';

import 'package:app/core/network/interceptors/auth_interceptor.dart';
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';

const _apiHost = 'api.example.com';

class _Tokens implements TokenStorage {
  int refreshes = 0;

  @override
  String? accessToken = 'secret';

  @override
  Future<String> refresh() async {
    refreshes++;
    return accessToken = 'fresh';
  }
}

/// Answers every retry with 200, and records what it was sent.
class _Server implements HttpClientAdapter {
  final seen = <RequestOptions>[];

  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    seen.add(options);
    return ResponseBody.fromString('{}', 200);
  }

  @override
  void close({bool force = false}) {}
}

class _Passed extends RequestInterceptorHandler {
  RequestOptions? options;

  @override
  void next(RequestOptions requestOptions) => options = requestOptions;
}

class _Failed extends ErrorInterceptorHandler {
  DioException? error;
  Response<dynamic>? resolved;

  @override
  void next(DioException err) => error = err;

  @override
  void resolve(Response<dynamic> response) => resolved = response;
}

DioException _unauthorized(String url) {
  final options = RequestOptions(path: url);
  return DioException(
    requestOptions: options,
    response: Response<dynamic>(requestOptions: options, statusCode: 401),
    type: DioExceptionType.badResponse,
  );
}

void main() {
  late _Tokens tokens;
  late _Server server;
  late AuthInterceptor interceptor;

  setUp(() {
    tokens = _Tokens();
    server = _Server();
    interceptor = AuthInterceptor(
      tokens,
      Dio()..httpClientAdapter = server,
      _apiHost,
    );
  });

  const thirdParty = [
    'https://cdn.example.net/image.png',
    'https://api.example.com.evil.example/steal',
    'https://api.example.com@evil.example/steal',
    'https://evil.example/?next=https://api.example.com',
    'https://evil.example/api.example.com',
  ];

  for (final url in thirdParty) {
    test('$url never gets the bearer token', () {
      final handler = _Passed();
      interceptor.onRequest(RequestOptions(path: url), handler);

      expect(handler.options!.headers, isNot(contains('Authorization')));
    });

    test('a 401 from $url triggers no refresh', () async {
      final handler = _Failed();
      await interceptor.onError(_unauthorized(url), handler);

      expect(tokens.refreshes, 0);
      expect(server.seen, isEmpty);
      expect(handler.error, isNotNull);
    });
  }

  test('your own API gets the token, and its 401 refreshes once', () async {
    const url = 'https://$_apiHost/orders';
    final passed = _Passed();
    interceptor.onRequest(RequestOptions(path: url), passed);
    expect(passed.options!.headers['Authorization'], 'Bearer secret');

    final failed = _Failed();
    final error = _unauthorized(url);
    error.requestOptions.headers['Authorization'] = 'Bearer secret';
    await interceptor.onError(error, failed);

    expect(tokens.refreshes, 1);
    expect(server.seen.single.headers['Authorization'], 'Bearer fresh');
    expect(failed.resolved?.statusCode, 200);
  });
}
```
