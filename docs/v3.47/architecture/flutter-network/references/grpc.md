---
title: gRPC with the grpc package
author: MOZGII, Inc.
lang: en
tags: [flutter, grpc, protobuf, streaming]
description: A secure gRPC channel, auth that covers unary and streaming calls, GrpcError mapping, bidirectional streaming and the gRPC-Web fallback built into the grpc package.
---
# gRPC With the grpc Package

gRPC gives a Flutter app generated clients and real streaming over HTTP/2.

```yaml
dependencies:
  grpc: ^5.2.0
  protobuf: ^6.1.0
```

## Generating Dart From Proto Files

`protoc` needs the Dart plugin. By convention, proto files live at `proto/services/*.proto`, and the output goes to `lib/core/grpc/generated/`. A file `chat.proto` produces `chat.pb.dart`, `chat.pbenum.dart`, `chat.pbjson.dart` and `chat.pbgrpc.dart`, and the last one re-exports the first.

```bash
# Activate protoc-gen-dart
dart pub global activate protoc_plugin

# Generate Dart code from proto files
protoc \
  --dart_out=grpc:lib/core/grpc/generated \
  -I proto \
  proto/services/chat.proto
```

This service is the one the examples below use.

```proto
// proto/services/chat.proto
syntax = "proto3";

package chat.v1;

message ChatMessage {
  string id = 1;
  string room_id = 2;
  string text = 3;
}

service ChatService {
  rpc ChatStream(stream ChatMessage) returns (stream ChatMessage);
}
```

## GrpcClient and the Secure Channel

A `ClientChannel` holds one HTTP/2 connection, so it's a singleton. `ChannelCredentials.secure()` is TLS, and `connectionTimeout` and `idleTimeout` are worth setting so a phone doesn't keep a dead connection open.

```dart
// lib/core/grpc/grpc_client.dart
import 'package:grpc/grpc.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class GrpcClient {
  GrpcClient()
    : channel = ClientChannel(
        const String.fromEnvironment('GRPC_HOST'),
        port: int.parse(
          const String.fromEnvironment('GRPC_PORT', defaultValue: '443'),
        ),
        options: const ChannelOptions(
          credentials: ChannelCredentials.secure(),
          connectionTimeout: Duration(seconds: 10),
          idleTimeout: Duration(minutes: 5),
        ),
      );

  final ClientChannel channel;

  Future<void> shutdown() => channel.shutdown();
}
```

## Auth Metadata for Unary and Streaming Calls

Reading a token from secure storage is async, and a gRPC interceptor can't be. `interceptUnary` must return a `ResponseFuture`, and an `async` method returns a plain `Future`, so that code doesn't compile.

The answer is a metadata provider. `CallOptions(providers: [...])` takes functions that run for every RPC, may be async, and can edit the outgoing metadata. Merge one in from both interceptor methods, and streaming calls carry auth too.

```dart
// lib/core/grpc/auth_client_interceptor.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:grpc/grpc.dart';
import 'package:injectable/injectable.dart';

@injectable
class AuthClientInterceptor extends ClientInterceptor {
  AuthClientInterceptor(this._storage);

  final FlutterSecureStorage _storage;

  Future<void> _addToken(Map<String, String> metadata, String uri) async {
    final token = await _storage.read(key: 'access_token');
    if (token != null) metadata['authorization'] = 'Bearer $token';
  }

  CallOptions _withAuth(CallOptions options) =>
      options.mergedWith(CallOptions(providers: [_addToken]));

  @override
  ResponseFuture<R> interceptUnary<Q, R>(
    ClientMethod<Q, R> method,
    Q request,
    CallOptions options,
    ClientUnaryInvoker<Q, R> invoker,
  ) => invoker(method, request, _withAuth(options));

  @override
  ResponseStream<R> interceptStreaming<Q, R>(
    ClientMethod<Q, R> method,
    Stream<Q> requests,
    CallOptions options,
    ClientStreamingInvoker<Q, R> invoker,
  ) => invoker(method, requests, _withAuth(options));
}
```

Providers run once, when a call starts. A stream that outlives its token keeps going until the server ends it with `unauthenticated`, so treat that status as the signal to refresh and reconnect. `flutter-database` has a reconnect loop for this.

## Error Mapping: GrpcError to Failure

`GrpcError.code` is a plain `int`, and `StatusCode` is a class of `int` constants. There's no `code.value`. Those constants work directly as switch patterns.

```dart
// lib/core/grpc/grpc_error_mapper.dart
import 'package:grpc/grpc.dart';

import '../errors/failure.dart';

abstract final class GrpcErrorMapper {
  static Failure map(GrpcError error) => switch (error.code) {
    StatusCode.unauthenticated => Failure.auth(
      error.message ?? 'Unauthenticated',
    ),
    StatusCode.permissionDenied => Failure.auth(
      error.message ?? 'Permission denied',
    ),
    StatusCode.notFound => Failure.server(404, error.message ?? 'Not found'),
    StatusCode.unavailable => Failure.network(
      error.message ?? 'Service unavailable',
    ),
    StatusCode.deadlineExceeded => const Failure.network('Request timed out'),
    _ => Failure.server(error.code, error.message ?? 'gRPC error'),
  };
}
```

## Bidirectional Streaming: A Chat Example

The generated client takes a stream of outgoing messages and returns a stream of incoming ones. Pass the auth interceptor to the client's constructor, and every call through it is covered. The file below imports `chat.pbgrpc.dart`, which re-exports the message classes.

```dart
// lib/features/chat/data/datasources/chat_grpc_datasource.dart
import 'package:grpc/grpc.dart';

import '../../../../core/grpc/generated/services/chat.pbgrpc.dart';

class ChatGrpcDatasource {
  ChatGrpcDatasource(ClientChannel channel, ClientInterceptor auth)
    : _stub = ChatServiceClient(channel, interceptors: [auth]);

  final ChatServiceClient _stub;

  /// Returns a stream of incoming messages. The caller sends through [outgoing].
  Stream<ChatMessage> openChatStream({required Stream<ChatMessage> outgoing}) =>
      _stub.chatStream(outgoing);
}
```

## Transport-Agnostic Repository Interface

Business logic in use cases and providers never knows which transport is active. The domain declares the contract, and a repository implements it with a gRPC or a REST data source.

```dart
// lib/features/chat/domain/repositories/chat_repository.dart
import 'package:fpdart/fpdart.dart';

import '../../../../core/errors/failure.dart';

class ChatEntry {
  const ChatEntry({required this.id, required this.text});
  final String id;
  final String text;
}

abstract interface class ChatRepository {
  Stream<ChatEntry> openStream({required String roomId});
  Future<Either<Failure, Unit>> sendMessage(ChatEntry message);
}
```

## gRPC-Web Fallback Without Another Package

gRPC-Web is part of the `grpc` package, under `package:grpc/grpc_web.dart`, and there's no separate dependency to add. Use it when corporate proxies block gRPC on port 443. It needs a translating proxy such as Envoy in front of your server, and it has no client streaming, so a bidirectional call like the one above won't work over it.

```dart
// lib/core/grpc/grpc_web_channel.dart
import 'package:grpc/grpc_web.dart';

final channel = GrpcWebClientChannel.xhr(Uri.parse('https://api.example.com'));
```

Flutter web can only use gRPC-Web. For code that runs on both, `GrpcOrGrpcWebClientChannel` from `package:grpc/grpc_or_grpcweb.dart` picks the right channel per platform.
