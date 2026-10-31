---
title: GraphQL with graphql_flutter
author: MOZGII, Inc.
lang: en
tags: [flutter, graphql, subscriptions, hive]
description: A GraphQL client with split HTTP and WebSocket links, authenticated subscriptions, query, mutation and subscription patterns, and error mapping.
---
# GraphQL With graphql_flutter

`graphql_flutter` gives you a normalised cache and subscriptions. It stores that cache through `hive_ce` as of version 5.3.

```yaml
dependencies:
  graphql_flutter: ^5.3.0
  hive_ce_flutter: ^2.5.0  # graphql_flutter 5.3 stores its cache with hive_ce
```

## GraphQL Client Setup With Split Links

`Link.split` sends subscriptions over the WebSocket link and everything else over HTTP. `AuthLink` adds a header to HTTP requests, and a WebSocket has no per-message headers, so the socket authenticates in `initialPayload`. That function runs on each connection, which means a reconnect picks up a fresh token.

`HiveStore()` needs Hive to be ready. Call `await initHiveForFlutter();` in `main()` before building the client, or the first cache read fails.

```dart
// lib/core/graphql/graphql_client.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:graphql_flutter/graphql_flutter.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class AppGraphQLClient {
  AppGraphQLClient(this._storage) {
    final authLink = AuthLink(getToken: _bearer);

    final httpLink = HttpLink(const String.fromEnvironment('GRAPHQL_URL'));

    final wsLink = WebSocketLink(
      const String.fromEnvironment('GRAPHQL_WS_URL'),
      config: SocketClientConfig(
        inactivityTimeout: const Duration(seconds: 30),
        initialPayload: () async => {'Authorization': await _bearer()},
      ),
    );

    final splitLink = Link.split(
      (request) => request.isSubscription,
      wsLink,
      authLink.concat(httpLink),
    );

    client = GraphQLClient(
      link: splitLink,
      cache: GraphQLCache(store: HiveStore()),
    );
  }

  late final GraphQLClient client;
  final FlutterSecureStorage _storage;

  Future<String?> _bearer() async {
    final token = await _storage.read(key: 'access_token');
    return token != null ? 'Bearer $token' : null;
  }
}
```

`autoReconnect` is on by default in `SocketClientConfig`.

## Query, Mutation and Subscription Patterns

A query can return the cached result and refresh it with `FetchPolicy.cacheAndNetwork`. Always check `hasException` before reading `data`. A subscription returns a `Stream` of results.

```dart
// lib/features/users/data/datasources/user_graphql_datasource.dart
import 'package:fpdart/fpdart.dart';
import 'package:graphql_flutter/graphql_flutter.dart';

import '../../../../core/errors/failure.dart';
import '../../../../core/graphql/graphql_errors.dart';

class UserGraphqlDatasource {
  const UserGraphqlDatasource(this._client);
  final GraphQLClient _client;

  Future<Either<Failure, Map<String, dynamic>>> getUser(String userId) async {
    final result = await _client.query(
      QueryOptions(
        document: gql(r'''
          query GetUser($id: ID!) {
            user(id: $id) { id email displayName }
          }
        '''),
        variables: {'id': userId},
        fetchPolicy: FetchPolicy.cacheAndNetwork,
      ),
    );

    if (result.hasException) return left(mapGraphQLError(result.exception!));
    return right(result.data!['user'] as Map<String, dynamic>);
  }

  Future<Either<Failure, Unit>> updateProfile(
    Map<String, dynamic> input,
  ) async {
    final result = await _client.mutate(
      MutationOptions(
        document: gql(
          r'mutation UpdateProfile($input: UpdateProfileInput!) { updateProfile(input: $input) { id } }',
        ),
        variables: {'input': input},
      ),
    );
    return result.hasException
        ? left(mapGraphQLError(result.exception!))
        : right(unit);
  }

  Stream<QueryResult<Object?>> watchMessages(String roomId) =>
      _client.subscribe(
        SubscriptionOptions(
          document: gql(
            r'subscription OnMessage($roomId: ID!) { message(roomId: $roomId) { id text } }',
          ),
          variables: {'roomId': roomId},
        ),
      );
}
```

## GraphQL Error Mapping

A failed operation has either a `linkException`, meaning the transport failed, or `graphqlErrors` from the server, which may carry a code in `extensions`. A GraphQL server can answer 200 with errors inside, so this check can't be skipped.

```dart
// lib/core/graphql/graphql_errors.dart
import 'package:graphql_flutter/graphql_flutter.dart';

import '../errors/failure.dart';

Failure mapGraphQLError(OperationException exception) {
  final link = exception.linkException;
  if (link != null) return Failure.network(link.toString());

  final errors = exception.graphqlErrors;
  if (errors.isNotEmpty) {
    final code = errors.first.extensions?['code'] as String? ?? 'UNKNOWN';
    return switch (code) {
      'UNAUTHENTICATED' => Failure.auth(errors.first.message),
      _ => Failure.server(0, errors.first.message),
    };
  }
  return const Failure.network('Unknown GraphQL error');
}
```
