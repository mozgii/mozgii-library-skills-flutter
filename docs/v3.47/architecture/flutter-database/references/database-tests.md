---
title: Tests for storage and sync
author: MOZGII, Inc.
lang: en
tags: [flutter, testing, drift, grpc, sync]
description: Device-free tests that run the Drift schema on in-memory SQLite and prove a version upgrade, enum storage, sync queue order, parked failures, conflict detection and a reconnect loop that always ends.
---
# Tests for Storage and Sync

These run under `flutter test` with no device. Drift's in-memory database is real SQLite, so a passing test means the schema and queries work. Imports use the package name `app`: replace it with yours.

## Drift Schema, Enum Storage and the Sync Queue

The first test reads the raw column to prove what `textEnum` stores. The queue tests prove order, that a network failure stops the drain with the failed item still pending, that a permanent failure is parked without stopping it, that a write made during a drain goes out in that drain, and that `start()` drains writes left from the last run. The repository tests prove a write goes out right after its transaction, with no reconnect, that the background sync leaves unsent work alone, and that a slow response can't bring back a post deleted while it was on its way.

```dart
// test/core/database/app_database_test.dart
import 'dart:async';

import 'package:app/core/database/app_database.dart';
import 'package:app/core/sync/conflict_policies.dart' show isConflicted;
import 'package:app/core/sync/sync_service.dart';
import 'package:app/features/posts/data/repositories/post_repository_impl.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:drift/drift.dart' hide isNotNull, isNull;
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';

class _FakeRemote implements PostRemoteDataSource {
  final created = <String>[];

  RemotePost _post(String id) => RemotePost(
    id: id,
    title: 'Remote $id',
    body: 'Body',
    authorId: 'a',
    updatedAt: DateTime.utc(2026),
  );

  @override
  Future<List<RemotePost>> getPosts() async => ['1', '2', '3'].map(_post).toList();

  @override
  Future<void> createPost(PostsTableData post) async => created.add(post.id);

  @override
  Future<void> deletePost(String id) async {}
}

/// Answers with what the server held before [duringFetch] ran.
class _SlowRemote extends _FakeRemote {
  _SlowRemote(this.duringFetch);

  final Future<void> Function() duringFetch;

  @override
  Future<List<RemotePost>> getPosts() async {
    final response = await super.getPosts();
    await duringFetch();
    return response;
  }
}

class _Quiet implements Connectivity {
  @override
  Stream<List<ConnectivityResult>> get onConnectivityChanged =>
      const Stream.empty();

  @override
  dynamic noSuchMethod(Invocation invocation) => super.noSuchMethod(invocation);
}

void main() {
  late AppDatabase db;

  setUp(() => db = AppDatabase(NativeDatabase.memory()));
  tearDown(() => db.close());

  Future<void> insertPost() => db.postsDao.upsertPost(
    PostsTableCompanion.insert(
      id: '1',
      title: 'Title',
      body: 'Body',
      authorId: 'a',
    ),
  );

  test('textEnum stores the name and reads back the enum', () async {
    await insertPost();
    final raw = await db
        .customSelect('SELECT sync_status FROM posts_table')
        .getSingle();
    expect(raw.read<String>('sync_status'), 'localOnly');

    final row = (await db.postsDao.getAllPosts()).single;
    expect(row.syncStatus, SyncStatus.localOnly);
  });

  test('a conflict needs both sides to move past the last sync', () async {
    await insertPost();
    final row = (await db.postsDao.getAllPosts()).single;
    final lastSync = DateTime.utc(2026, 1, 1);
    final after = DateTime.utc(2026, 1, 2);

    final local = row.copyWith(
      updatedAt: after,
      serverUpdatedAt: Value(lastSync),
    );
    RemotePost remoteAt(DateTime time) => RemotePost(
      id: '1',
      title: 'T',
      body: 'B',
      authorId: 'a',
      updatedAt: time,
    );

    expect(isConflicted(local, remoteAt(after)), isTrue);
    expect(isConflicted(local, remoteAt(lastSync)), isFalse);
    expect(isConflicted(row, remoteAt(after)), isFalse); // never synced
  });

  group('sync queue', () {
    Future<void> enqueue(String payload) async => db.syncQueueDao.enqueue(
      SyncOperationsTableCompanion.insert(
        operation: 'post.delete',
        payload: payload,
      ),
    );

    test('drains in order and marks each one complete', () async {
      await enqueue('a');
      await enqueue('b');
      final sent = <String>[];
      await SyncService(
        db.syncQueueDao,
        Connectivity(),
        (op) async => sent.add(op.payload),
      ).drainQueue();

      expect(sent, ['a', 'b']);
      expect(await db.syncQueueDao.getPending(), isEmpty);
    });

    test('a failure stops the drain and keeps the operation', () async {
      await enqueue('a');
      await enqueue('b');
      final sent = <String>[];
      await SyncService(db.syncQueueDao, Connectivity(), (op) async {
        sent.add(op.payload);
        throw Exception('offline');
      }).drainQueue();

      expect(sent, ['a']);
      final pending = await db.syncQueueDao.getPending();
      expect(pending.map((o) => o.payload), ['a', 'b']);
      expect(pending.first.attempts, 1);
    });

    test('a permanent failure is parked and the drain goes on', () async {
      await enqueue('a');
      await enqueue('b');
      final sent = <String>[];
      await SyncService(db.syncQueueDao, Connectivity(), (op) async {
        sent.add(op.payload);
        if (op.payload == 'a') throw const PermanentSyncFailure('refused');
      }).drainQueue();

      expect(sent, ['a', 'b']);
      expect(await db.syncQueueDao.getPending(), isEmpty);
      final parked = await db.syncQueueDao.watchParked().first;
      expect(parked.single.payload, 'a');
    });

    test('a write made during a drain goes out in that drain', () async {
      await enqueue('a');
      final sent = <String>[];
      late final SyncService service;
      service = SyncService(db.syncQueueDao, Connectivity(), (op) async {
        sent.add(op.payload);
        if (op.payload == 'a') {
          await enqueue('b');
          await service.drainQueue(); // returns at once: a drain is running
        }
      });
      await service.drainQueue();

      expect(sent, ['a', 'b']);
      expect(await db.syncQueueDao.getPending(), isEmpty);
    });

    test('start drains what the last run left behind', () async {
      await enqueue('a');
      final sent = Completer<String>();
      final service = SyncService(
        db.syncQueueDao,
        _Quiet(),
        (op) async => sent.complete(op.payload),
      )..start();

      expect(await sent.future, 'a');
      await service.stop();
    });
  });

  test('a write goes out after its transaction, with no reconnect', () async {
    final remote = _FakeRemote();
    final sync = SyncService(db.syncQueueDao, _Quiet(), postSender(db, remote));
    final repository = PostRepositoryImpl(db, remote, sync.drainQueue);

    await repository.createPost(
      const Post(id: '9', title: 'Title', body: 'Body'),
      'a',
    );
    await db
        .select(db.syncOperationsTable)
        .watch()
        .firstWhere((ops) => ops.single.status == 'completed');

    expect(remote.created, ['9']);
  });

  test('the background sync leaves unsent work alone', () async {
    await insertPost(); // localOnly: not sent yet
    await db.postsDao.upsertPost(
      PostsTableCompanion.insert(
        id: '2',
        title: 'Title',
        body: 'Body',
        authorId: 'a',
        syncStatus: const Value(SyncStatus.synced),
      ),
    );
    await db.postsDao.upsertPost(
      PostsTableCompanion.insert(
        id: '3',
        title: 'Old',
        body: 'Body',
        authorId: 'a',
        syncStatus: const Value(SyncStatus.synced),
      ),
    );
    // No drain: the delete stays queued, as it does offline.
    final repository = PostRepositoryImpl(db, _FakeRemote(), () async {});

    await repository.deletePost('2');
    await repository.getPosts();
    await db.postsDao.watchAllPosts().firstWhere(
      (rows) => rows.any((row) => row.title == 'Remote 3'),
    );

    final rows = await db.postsDao.getAllPosts();
    expect(rows.map((r) => r.id), unorderedEquals(['1', '3']));
    expect(rows.firstWhere((r) => r.id == '1').title, 'Title');
  });

  test('a delete finished during the fetch stays deleted', () async {
    await db.postsDao.upsertPost(
      PostsTableCompanion.insert(
        id: '2',
        title: 'Title',
        body: 'Body',
        authorId: 'a',
        syncStatus: const Value(SyncStatus.synced),
      ),
    );
    late final PostRepositoryImpl repository;
    late final SyncService sync;
    final remote = _SlowRemote(() async {
      await repository.deletePost('2');
      await sync.drainQueue(); // the delete reaches the server, and completes
    });
    sync = SyncService(db.syncQueueDao, _Quiet(), postSender(db, remote));
    repository = PostRepositoryImpl(db, remote, () async {});

    await repository.getPosts();
    await db.postsDao.watchAllPosts().firstWhere(
      (rows) => rows.any((row) => row.title == 'Remote 3'),
    );

    expect(await db.syncQueueDao.getPending(), isEmpty);
    expect(await db.postsDao.getById('2'), isNull);
  });

  test('lastTimestamp reads the newest message in a room', () async {
    Future<void> add(String id, String room, int time) => db.messagesDao.upsert(
      MessagesTableCompanion.insert(
        id: id,
        roomId: room,
        content: 'x',
        timestampMs: time,
      ),
    );
    await add('1', 'r1', 100);
    await add('2', 'r1', 200);
    await add('3', 'r2', 900);
    await add('2', 'r1', 200); // a replay must not duplicate

    expect(await db.messagesDao.lastTimestamp('r1'), 200);
    expect(await db.messagesDao.watchAllMessages('r1').first, hasLength(2));
  });
}
```

## The Upgrade From Version 1 to Version 2

A migration test starts from the old tables, not from `onCreate`, because `onCreate` never runs the upgrade path. `NativeDatabase.memory(setup: ...)` builds the version 1 schema by hand and sets `user_version`, so opening `AppDatabase` runs `onUpgrade`. This is the test that fails with `Cannot add a column with non-constant default` if `updatedAt` goes through `addColumn`. It also proves what the upgrade does to each row: one the server has comes out `synced`, so the background sync keeps refreshing it, and an unsent one stays `localOnly` with a `post.create` queue entry.

```dart
// test/core/database/migration_test.dart
import 'package:app/core/database/app_database.dart';
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  test('version 1 upgrades to 2 and keeps its rows', () async {
    final db = AppDatabase(
      NativeDatabase.memory(
        setup: (raw) {
          raw.execute('''
            CREATE TABLE posts_table (
              id TEXT NOT NULL PRIMARY KEY,
              title TEXT NOT NULL,
              body TEXT NOT NULL,
              author_id TEXT NOT NULL,
              created_at INTEGER NOT NULL DEFAULT (strftime('%s', CURRENT_TIMESTAMP)),
              is_synced INTEGER NOT NULL DEFAULT 0
            );
          ''');
          raw.execute(
            "INSERT INTO posts_table (id, title, body, author_id, is_synced) "
            "VALUES ('1', 'Kept', 'Body', 'a', 1), ('2', 'Draft', 'Body', 'a', 0);",
          );
          raw.userVersion = 1;
        },
      ),
    );
    addTearDown(db.close);

    final kept = (await db.postsDao.getById('1'))!;
    expect(kept.title, 'Kept');
    expect(kept.syncStatus, SyncStatus.synced);
    expect(kept.serverUpdatedAt, isNull);

    final draft = (await db.postsDao.getById('2'))!;
    expect(draft.syncStatus, SyncStatus.localOnly);
    final queued = (await db.syncQueueDao.getPending()).single;
    expect(queued.operation, 'post.create');
    expect(queued.payload, '2');
  });
}
```

For a released app, generate these steps from schema snapshots with `dart run drift_dev make-migrations`, which also writes a test per version.

## A Reconnect Loop That Always Ends

The dangerous failure of a stream is a loop that never stops. These tests inject `open`, so no server runs, and each has a timeout that fails it if the loop spins. They use a fake token provider that records every call, which proves a refresh really was requested. The last two prove that `cancel()` returns with no message due, and that it stops the retry delay.

```dart
// test/features/chat/grpc_chat_datasource_test.dart
import 'dart:async';

import 'package:app/core/grpc/generated/chat.pbgrpc.dart';
import 'package:app/features/chat/data/datasources/grpc_chat_datasource.dart';
import 'package:fixnum/fixnum.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:grpc/grpc.dart';

void main() {
  final channel = ClientChannel('localhost');
  final refreshes = <bool>[];

  Future<String> token({bool forceRefresh = false}) async {
    refreshes.add(forceRefresh);
    return forceRefresh ? 'fresh' : 'old';
  }

  GrpcChatDataSource source(OpenStream open) => GrpcChatDataSource(
    stub: ChatServiceClient(channel),
    tokenProvider: token,
    open: open,
  );

  setUp(refreshes.clear);
  tearDownAll(channel.shutdown);

  test('unauthenticated forces one real refresh, then reconnects', () async {
    final tokens = <String?>[];
    final messages = await source((request, options) {
      tokens.add(options.metadata['authorization']);
      return tokens.length == 1
          ? Stream.error(GrpcError.unauthenticated('expired'))
          : Stream.value(Message(id: 'm1', timestampMs: Int64(5)));
    }).watchRoom('room', 0).toList().timeout(const Duration(seconds: 2));

    expect(messages.single.id, 'm1');
    expect(refreshes, [false, true]);
    expect(tokens, ['Bearer old', 'Bearer fresh']);
  });

  test('a refused fresh token ends the stream with an error', () async {
    var opens = 0;
    final stream = source((request, options) {
      opens++;
      return Stream.error(GrpcError.unauthenticated('dead session'));
    }).watchRoom('room', 0);

    await expectLater(
      stream.toList().timeout(const Duration(seconds: 2)),
      throwsA(isA<GrpcError>()),
    );
    expect(opens, 2);
  });

  test('cancelled ends the stream without an error', () async {
    final messages = await source(
      (request, options) => Stream.error(GrpcError.cancelled('left the room')),
    ).watchRoom('room', 0).toList().timeout(const Duration(seconds: 2));
    expect(messages, isEmpty);
  });

  test('a reconnect resumes after the last message', () async {
    final since = <Int64>[];
    final messages = await source((request, options) async* {
      since.add(request.sinceTimestampMs);
      if (since.length == 1) {
        yield Message(id: 'm1', timestampMs: Int64(42));
        throw GrpcError.unavailable('network dropped');
      }
      yield Message(id: 'm2', timestampMs: Int64(43));
    }).watchRoom('room', 7).toList().timeout(const Duration(seconds: 3));

    expect(messages.map((m) => m.id), ['m1', 'm2']);
    expect(since, [Int64(7), Int64(42)]);
  });

  test('cancel returns in a silent room and stops the call', () async {
    var callStopped = false;
    final silent = StreamController<Message>(
      onCancel: () => callStopped = true,
    );
    final subscription = source(
      (request, options) => silent.stream,
    ).watchRoom('room', 0).listen((_) {});
    await pumpEventQueue();

    await subscription.cancel().timeout(const Duration(seconds: 2));
    expect(callStopped, isTrue);
  });

  test('cancel stops the retry delay while the server is down', () async {
    var opens = 0;
    final subscription = source((request, options) {
      opens++;
      return Stream.error(GrpcError.unavailable('server down'));
    }).watchRoom('room', 0).listen((_) {});
    await pumpEventQueue();

    await subscription.cancel().timeout(const Duration(seconds: 2));
    await Future<void>.delayed(const Duration(milliseconds: 1100));
    expect(opens, 1);
  });
}
```
