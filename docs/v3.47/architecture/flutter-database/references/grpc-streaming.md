---
title: gRPC streaming into Drift
author: MOZGII, Inc.
lang: en
tags: [flutter, grpc, streaming, drift, offline]
description: A reconnecting server stream with real token refresh and bounded retries, Riverpod bridging, error mapping, and Drift as the source of truth for live messages.
---
# gRPC Streaming Into Drift

gRPC streaming gives real-time data without WebSockets or Realtime polling. Use it for chat messages, live feeds, collaborative editing and real-time notifications. For high-throughput scenarios, server streaming replaces Supabase Realtime.

```yaml
dependencies:
  fixnum: ^1.1.1
  grpc: ^5.2.0
  protobuf: ^6.1.0
```

## Proto Definition for a Chat Stream

`since_timestamp_ms` lets a reconnect resume from the last known message.

```proto
// proto/chat.proto
syntax = "proto3";

package chat.v1;

message Message {
  string id = 1;
  string room_id = 2;
  string content = 3;
  int64 timestamp_ms = 4;
  string author_id = 5;
}

message WatchRoomRequest {
  string room_id = 1;
  int64 since_timestamp_ms = 2; // for resume from last known message
}

service ChatService {
  rpc WatchRoom(WatchRoomRequest) returns (stream Message);
  rpc SendMessage(Message) returns (Message);
}
```

Run `protoc` with the Dart plugin to generate code. `protoc_plugin` isn't a project dependency: `protoc` looks for its `protoc-gen-dart` on your `PATH`, so activate it once, at the version these examples were generated with:

```bash
dart pub global activate protoc_plugin 25.1.0
export PATH="$PATH:$HOME/.pub-cache/bin"
protoc --dart_out=grpc:lib/core/grpc/generated -I proto proto/chat.proto
```

This writes `chat.pb.dart`, `chat.pbenum.dart`, `chat.pbjson.dart` and `chat.pbgrpc.dart`. Files are named after `chat.proto`, not after `package chat.v1`, and `chat.pbgrpc.dart` re-exports `chat.pb.dart`, so one import is enough. A proto3 `int64` field is an `Int64` from `package:fixnum`, so wrap with `Int64(value)` and read with `.toInt()`.

## A gRPC Data Source With Real Token Refresh

`watchRoom` returns a `StreamController`'s stream, and its `onCancel` stops the open call and the retry delay. So `cancel()` returns at once, even in a silent room or while the server is down. An `async*` loop can't do that: a cancel only reaches it at its next `yield`, so with no message coming `cancel()` hangs and the reconnects go on.

`recover` handles each way a call can end:

- **`cancelled`:** a user-initiated cancel, so stop without a retry.
- **`unauthenticated`:** ask for a new token with `forceRefresh: true`, and try again once. A second `unauthenticated` straight after a refresh means the session is dead, so the error goes to the caller. Waiting and re-sending the same token would loop forever.
- **`unavailable` and `deadlineExceeded`:** exponential backoff from 1 to 30 seconds. A delivered message resets the backoff and the refresh allowance.
- **Anything else:** a real error, such as `permissionDenied`, goes to the caller.

`since` moves forward with each message, so a reconnect resumes where the stream stopped instead of replaying from the start. `open` is a function, so a test can drive the reconnects with no server.

```dart
// lib/features/chat/data/datasources/grpc_chat_datasource.dart
import 'dart:async';

import 'package:fixnum/fixnum.dart';
import 'package:grpc/grpc.dart';

import '../../../../core/grpc/generated/chat.pbgrpc.dart';

abstract interface class ChatRemoteDataSource {
  Stream<Message> watchRoom(String roomId, int sinceTimestampMs);
  Future<Message> sendMessage(Message message);
}

/// Returns a valid access token, or refreshes first when [forceRefresh] is set.
typedef TokenProvider = Future<String> Function({bool forceRefresh});

typedef OpenStream = Stream<Message> Function(
  WatchRoomRequest request,
  CallOptions options,
);

class GrpcChatDataSource implements ChatRemoteDataSource {
  GrpcChatDataSource({
    required ChatServiceClient stub,
    required TokenProvider tokenProvider,
    OpenStream? open,
  }) : _stub = stub,
       _tokenProvider = tokenProvider,
       _open =
           open ??
           ((request, options) => stub.watchRoom(request, options: options));

  final ChatServiceClient _stub;
  final TokenProvider _tokenProvider;
  final OpenStream _open;

  static const _retryable = {
    StatusCode.unavailable,
    StatusCode.deadlineExceeded,
  };

  Future<CallOptions> _options({bool forceRefresh = false}) async {
    final token = await _tokenProvider(forceRefresh: forceRefresh);
    return CallOptions(metadata: {'authorization': 'Bearer $token'});
  }

  @override
  Stream<Message> watchRoom(String roomId, int sinceTimestampMs) {
    var since = sinceTimestampMs;
    var delayMs = 1000;
    var refreshed = false;
    var stopped = false;
    Stream<Message>? responses;
    StreamSubscription<Message>? call;
    Timer? delay;
    late final StreamController<Message> controller;

    Future<void> connect() async {
      void recover(Object error, StackTrace stack) {
        if (stopped) return;
        if (error is GrpcError) {
          if (error.code == StatusCode.cancelled) {
            unawaited(controller.close());
            return;
          }
          if (error.code == StatusCode.unauthenticated && !refreshed) {
            refreshed = true;
            unawaited(connect());
            return;
          }
          if (_retryable.contains(error.code)) {
            delay = Timer(Duration(milliseconds: delayMs), connect);
            delayMs = (delayMs * 2).clamp(1000, 30000);
            return;
          }
        }
        controller.addError(error, stack); // includes a refused fresh token
        unawaited(controller.close());
      }

      try {
        final options = await _options(forceRefresh: refreshed);
        if (stopped) return; // cancelled while the token was on its way
        final request = WatchRoomRequest(
          roomId: roomId,
          sinceTimestampMs: Int64(since),
        );
        final stream = responses = _open(request, options);
        final subscription = stream.listen(
          (message) {
            since = message.timestampMs.toInt();
            delayMs = 1000;
            refreshed = false;
            controller.add(message);
          },
          onError: recover,
          onDone: controller.close, // the server finished the stream
          cancelOnError: true,
        );
        if (controller.isPaused) subscription.pause();
        call = subscription;
      } on Object catch (error, stack) {
        recover(error, stack);
      }
    }

    controller = StreamController<Message>(
      onListen: connect,
      onPause: () => call?.pause(),
      onResume: () => call?.resume(),
      onCancel: () async {
        stopped = true;
        delay?.cancel();
        // A cancelled listener can't stop a call that is still connecting.
        if (responses case final ResponseStream<Message> open) {
          await open.cancel();
        }
        await call?.cancel();
      },
    );
    return controller.stream;
  }

  @override
  Future<Message> sendMessage(Message message) async =>
      _stub.sendMessage(message, options: await _options());
}
```

The `ClientChannel` behind `stub` is shared by every call, so this class doesn't close it: build one with `ChannelOptions(credentials: ChannelCredentials.secure())` and shut it down once, at app exit.

## Switching Transports With Injectable Environments

The domain repository doesn't know about gRPC or REST: both implementations satisfy `ChatRemoteDataSource`, and injectable's `env` picks one, so switching transport leaves the domain untouched. Run with `getIt.init(environment: Environment.prod)`.

```dart
// lib/core/di/chat_bindings.dart
import 'package:grpc/grpc.dart';
import 'package:injectable/injectable.dart';

import '../../features/chat/data/datasources/grpc_chat_datasource.dart';
import '../grpc/generated/chat.pbgrpc.dart';

abstract interface class ChatSession {
  Future<String> token({bool forceRefresh});
}

@LazySingleton(as: ChatRemoteDataSource, env: [Environment.prod])
class GrpcChatDataSourceImpl extends GrpcChatDataSource {
  GrpcChatDataSourceImpl(ClientChannel channel, ChatSession session)
    : super(stub: ChatServiceClient(channel), tokenProvider: session.token);
}

@LazySingleton(as: ChatRemoteDataSource, env: [Environment.dev])
class RestChatDataSourceImpl implements ChatRemoteDataSource {
  @override
  Stream<Message> watchRoom(String roomId, int sinceTimestampMs) =>
      throw UnimplementedError('Poll your REST endpoint here');

  @override
  Future<Message> sendMessage(Message message) =>
      throw UnimplementedError('Post to your REST endpoint here');
}
```

## Bridge: gRPC Stream to a Riverpod StreamProvider

The provider resumes from the last local timestamp. Riverpod cancels the stream when nothing watches it, which runs `onCancel`, so no `onDispose` is needed. Don't shut the channel down here: other screens share it.

```dart
// lib/features/chat/presentation/providers/chat_stream_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/grpc/generated/chat.pb.dart';
import '../../data/datasources/grpc_chat_datasource.dart';

part 'chat_stream_provider.g.dart';

@riverpod
ChatRemoteDataSource chatRemoteDataSource(Ref ref) =>
    throw UnimplementedError('Bind this to GetIt');

@riverpod
int lastMessageTimestamp(Ref ref, String roomId) => 0;

@riverpod
Stream<Message> chatMessages(Ref ref, String roomId) {
  final dataSource = ref.watch(chatRemoteDataSourceProvider);
  // Resume from the last known local timestamp.
  final since = ref.read(lastMessageTimestampProvider(roomId));
  return dataSource.watchRoom(roomId, since);
}
```

## Error Recovery: gRPC Status to the Failure Sealed Type

`error.code` is a plain `int`, and `StatusCode` holds `int` constants that work as switch patterns.

```dart
// lib/core/errors/grpc_error_mapper.dart
import 'package:grpc/grpc.dart';

import 'failure.dart';

Failure mapGrpcError(GrpcError error) => switch (error.code) {
  StatusCode.unavailable => const Failure.network('Service unavailable'),
  StatusCode.unauthenticated => const Failure.auth('Session expired'),
  StatusCode.cancelled => const Failure.network('Request cancelled'),
  StatusCode.resourceExhausted => const Failure.server(429, 'Rate limited'),
  StatusCode.internal => Failure.server(500, error.message ?? 'Server error'),
  _ => Failure.network(error.message ?? 'gRPC error'),
};
```

## Messages DAO: Reading the Last Timestamp

The DAO behind the offline pattern. `lastTimestamp` reads the newest time in one room, and `upsert` makes a replayed message harmless. `@DriftDatabase` lists it in `daos`, so `app_database.dart` imports this file.

```dart
// lib/core/database/messages_dao.dart
import 'package:drift/drift.dart';

import 'app_database.dart';

part 'messages_dao.g.dart';

@DriftAccessor(tables: [MessagesTable])
class MessagesDao extends DatabaseAccessor<AppDatabase>
    with _$MessagesDaoMixin {
  MessagesDao(super.db);

  Stream<List<MessagesTableData>> watchAllMessages(String roomId) =>
      (select(messagesTable)
            ..where((t) => t.roomId.equals(roomId))
            ..orderBy([(t) => OrderingTerm.asc(t.timestampMs)]))
          .watch();

  Future<int?> lastTimestamp(String roomId) {
    final latest = messagesTable.timestampMs.max();
    final query = selectOnly(messagesTable)
      ..addColumns([latest])
      ..where(messagesTable.roomId.equals(roomId));
    return query.map((row) => row.read(latest)).getSingle();
  }

  Future<void> upsert(MessagesTableCompanion message) =>
      into(messagesTable).insertOnConflictUpdate(message);
}
```

## gRPC + Drift Offline Pattern

Drift is the source of truth, and gRPC is the sync mechanism: the UI never reads from gRPC directly. Each arriving message is written to Drift, and the screen watches the Drift stream.

`WatchRoomRequest.since_timestamp_ms` is always populated from the last Drift record. Have the server answer with messages at or after that time, and reconnects are idempotent: messages already in Drift are skipped by the upsert, and none are lost. `watchRoom` passes errors it can't retry to its listener, such as a dead session or `permissionDenied`, so `startSync` reports them through `onError`.

```dart
// lib/features/chat/data/repositories/chat_repository_impl.dart
import 'dart:async';

import 'package:grpc/grpc.dart';

import '../../../../core/database/app_database.dart';
import '../../../../core/database/messages_dao.dart';
import '../../../../core/errors/failure.dart';
import '../../../../core/errors/grpc_error_mapper.dart';
import '../../../../core/grpc/generated/chat.pb.dart';
import '../datasources/grpc_chat_datasource.dart';

extension on Message {
  MessagesTableCompanion toDriftCompanion() => MessagesTableCompanion.insert(
    id: id,
    roomId: roomId,
    content: content,
    timestampMs: timestampMs.toInt(),
  );
}

abstract interface class ChatRepository {
  Stream<List<MessagesTableData>> watchMessages(String roomId);
  Future<void> startSync(String roomId);
  Future<void> stopSync();
  Future<void> reconnect(String roomId);
}

class ChatRepositoryImpl implements ChatRepository {
  ChatRepositoryImpl(this._grpc, this._dao, this._onSyncFailure);

  final ChatRemoteDataSource _grpc;
  final MessagesDao _dao;
  final void Function(Failure failure) _onSyncFailure;

  StreamSubscription<Message>? _grpcSub;

  /// The UI subscribes to this: a Drift stream, not gRPC.
  @override
  Stream<List<MessagesTableData>> watchMessages(String roomId) =>
      _dao.watchAllMessages(roomId);

  /// Call once per session to start sync.
  @override
  Future<void> startSync(String roomId) async {
    final since = await _dao.lastTimestamp(roomId) ?? 0;

    _grpcSub = _grpc
        .watchRoom(roomId, since)
        .listen(
          // Write arriving gRPC messages into Drift immediately.
          (proto) => unawaited(_dao.upsert(proto.toDriftCompanion())),
          // The stream ends on an error it can't retry: without this handler
          // that error is uncaught and the sync stops with no sign.
          onError: (Object error) => _onSyncFailure(
            error is GrpcError
                ? mapGrpcError(error)
                : Failure.network('Chat sync stopped: $error'),
          ),
        );
  }

  @override
  Future<void> stopSync() async {
    await _grpcSub?.cancel();
    _grpcSub = null;
  }

  /// On reconnect, resume from the last Drift timestamp: no messages lost.
  @override
  Future<void> reconnect(String roomId) async {
    await stopSync();
    await startSync(roomId);
  }
}
```
