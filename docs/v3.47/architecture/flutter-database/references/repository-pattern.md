---
title: The offline-first repository
author: MOZGII, Inc.
lang: en
tags: [flutter, repository, offline-first, drift]
description: A repository that returns local data at once, syncs in the background and enqueues remote writes, so the domain layer never sees storage or the network.
---
# The Offline-First Repository

The repository is the only contract between business logic and storage. Rows and remote models stop here, and the domain gets entities.

## Local First, Sync Behind

`getPosts` answers from Drift straight away and starts a sync it doesn't wait for. If the network is down, that sync fails silently, because the person already has data. New rows reach the screen through `watchPosts`, since the Drift stream fires on each write. `deletePost` shows the other half of the rule: change local storage now, and queue the remote call so it survives a restart.

Four details keep this correct. Each write and its queue entry share one `_db.transaction`, because Drift's transaction covers every DAO of the database, and a crash between two separate writes would lose the remote call or leave a queued call with no row. The background sync skips rows that still have unsent changes, ids with a pending `post.delete`, and every `post.delete` newer than the last operation completed before the fetch began, so a slow response can't bring back a deleted post or overwrite an edit, even when the delete finished while the response was on its way. `getPostById` asks Drift for one row rather than loading the table to search it. And after each transaction the repository starts a drain and doesn't wait for it: a connectivity event only fires on a change, so without this an online write would sit in the queue until the next reconnect or launch. Pass `SyncService.drainQueue` as the third constructor argument.

```dart
// lib/features/posts/data/repositories/post_repository_impl.dart
import 'dart:async';

import 'package:drift/drift.dart';
import 'package:fpdart/fpdart.dart';

import '../../../../core/database/app_database.dart';
import '../../../../core/errors/failure.dart';
import '../../../../core/sync/sync_service.dart';

class Post {
  const Post({required this.id, required this.title, required this.body});
  final String id;
  final String title;
  final String body;
}

abstract interface class PostRepository {
  Future<Either<Failure, List<Post>>> getPosts();
  Future<Either<Failure, Post>> getPostById(String id);
  Future<Either<Failure, Post>> createPost(Post post, String authorId);
  Future<Either<Failure, Unit>> deletePost(String id);
  Stream<List<Post>> watchPosts(); // live updates
}

class RemotePost {
  const RemotePost({
    required this.id,
    required this.title,
    required this.body,
    required this.authorId,
    required this.updatedAt,
  });

  final String id;
  final String title;
  final String body;
  final String authorId;
  final DateTime updatedAt;

  PostsTableCompanion toCompanion() => PostsTableCompanion.insert(
    id: id,
    title: title,
    body: body,
    authorId: authorId,
    updatedAt: Value(updatedAt),
    serverUpdatedAt: Value(updatedAt),
    syncStatus: const Value(SyncStatus.synced),
  );
}

abstract interface class PostRemoteDataSource {
  Future<List<RemotePost>> getPosts();
  Future<void> createPost(PostsTableData post);
  Future<void> deletePost(String id);
}

class PostRepositoryImpl implements PostRepository {
  PostRepositoryImpl(this._db, this._remote, this._drain);

  final AppDatabase _db;
  final PostRemoteDataSource _remote;
  final Future<void> Function() _drain;

  PostsDao get _posts => _db.postsDao;
  SyncQueueDao get _queue => _db.syncQueueDao;

  Post _toEntity(PostsTableData row) =>
      Post(id: row.id, title: row.title, body: row.body);

  @override
  Future<Either<Failure, List<Post>>> getPosts() async {
    // 1. Return local data immediately
    final local = await _posts.getAllPosts();
    // 2. Sync from remote in background
    unawaited(_syncFromRemote());
    return right(local.map(_toEntity).toList());
  }

  Future<void> _syncFromRemote() async {
    try {
      // A delete that finishes during the fetch is gone from the pending list
      // by the time this response, read before it, gets here.
      final sentBefore = await _queue.lastCompletedId();
      final remote = await _remote.getPosts();
      await _db.transaction(() async {
        // Work the server hasn't seen yet wins until the queue has sent it.
        final keepLocal = {
          ...await _posts.unsyncedIds(),
          ...await _queue.pendingPayloads('post.delete'),
          ...await _queue.payloadsAfter('post.delete', sentBefore),
        };
        await _posts.upsertPosts([
          for (final post in remote)
            if (!keepLocal.contains(post.id)) post.toCompanion(),
        ]);
      });
    } on Exception {
      // Sync failure is silent: local data was already returned.
    }
  }

  @override
  Future<Either<Failure, Post>> getPostById(String id) async {
    final row = await _posts.getById(id);
    return row == null
        ? left(Failure.cache('Post $id not found'))
        : right(_toEntity(row));
  }

  @override
  Future<Either<Failure, Post>> createPost(Post post, String authorId) async {
    await _db.transaction(() async {
      await _posts.upsertPost(
        PostsTableCompanion.insert(
          id: post.id,
          title: post.title,
          body: post.body,
          authorId: authorId,
        ),
      );
      await _queue.enqueue(
        SyncOperationsTableCompanion.insert(
          operation: 'post.create',
          payload: post.id,
        ),
      );
    });
    unawaited(_drain());
    return right(post);
  }

  @override
  Future<Either<Failure, Unit>> deletePost(String id) async {
    await _db.transaction(() async {
      await _posts.deletePost(id);
      await _queue.enqueue(
        SyncOperationsTableCompanion.insert(
          operation: 'post.delete',
          payload: id,
        ),
      );
    });
    unawaited(_drain());
    return right(unit);
  }

  @override
  Stream<List<Post>> watchPosts() =>
      _posts.watchAllPosts().map((rows) => rows.map(_toEntity).toList());
}

/// The function `SyncService` calls for each queued operation.
SendOperation postSender(AppDatabase db, PostRemoteDataSource remote) =>
    (op) async {
      switch (op.operation) {
        case 'post.create':
          final row = await db.postsDao.getById(op.payload);
          if (row == null) return; // deleted before it went out
          await remote.createPost(row);
          await db.postsDao.markSynced(row.id);
        case 'post.delete':
          await remote.deletePost(op.payload);
        default:
          throw PermanentSyncFailure('Unknown operation ${op.operation}');
      }
    };
```

`postSender` closes the loop: `markSynced` hands the row back to the background sync, which skips it until then. It also has to turn a refusal from your API into `PermanentSyncFailure`, so a request the server will never accept is parked instead of blocking the queue. A policy that flags conflicts passes the skipped rows to `ConflictResolver` rather than ignoring them (`references/conflict-policies.md`).

## One Failure Type for Storage and Sync

The repository, the gRPC data source and every caller report through this sealed type, so a screen handles one set of cases.

```dart
// lib/core/errors/failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failure.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network(String message) = NetworkFailure;
  const factory Failure.server(int statusCode, String message) = ServerFailure;
  const factory Failure.cache(String message) = CacheFailure;
  const factory Failure.auth(String message) = AuthFailure;
}
```
