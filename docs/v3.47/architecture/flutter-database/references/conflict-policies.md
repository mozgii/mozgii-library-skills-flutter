---
title: Sync conflict policies
author: MOZGII, Inc.
lang: en
tags: [flutter, sync, conflicts, drift]
description: Last write wins, server wins and flagging a conflict for the user, each written against Drift rows and real enum columns.
---
# Sync Conflict Policies

Choose one policy per entity type. These use the tables in the skill's `app_database.dart`, and take the same `RemotePost` the repository fetches, so the rows its background sync skips can go straight to `ConflictResolver`.

## Last Write Wins, the Server Wins Policy and the Conflict Flag

**Last write wins** is the simplest. Use it when data loss is acceptable, such as user profile fields and settings. It compares `updatedAt` timestamps, and those come from two clocks, so a phone set a day ahead always wins. Don't use it for anything you can't afford to lose.

**Server wins** suits fields the server owns: balance, subscription status, permissions. It always overwrites the local value for protected fields.

**Conflict flag + user resolution** is for data you can't lose, such as notes, documents and forms. A conflict exists when both `local.updatedAt` and the remote time moved past `serverUpdatedAt`, the last time the two agreed. A row that never synced has no shared point, so it isn't a conflict.

The repository doesn't tell the UI anything. It writes `SyncStatus.conflicted`, and the screen watches `PostsDao.watchConflicted`, a Drift stream, so the update arrives on its own.

```dart
// lib/core/sync/conflict_policies.dart
import 'package:drift/drift.dart';

import '../../features/posts/data/repositories/post_repository_impl.dart';
import '../database/app_database.dart';

class RemoteAccount {
  const RemoteAccount({
    required this.id,
    required this.displayName,
    required this.subscriptionTier,
    required this.accountBalance,
  });

  final String id;
  final String displayName;
  final String subscriptionTier;
  final int accountBalance;
}

abstract interface class PostRemote {
  Future<void> updatePost(PostsTableData local);
}

/// Detects a conflict: both sides changed since the last sync.
bool isConflicted(PostsTableData local, RemotePost remote) {
  final lastSync = local.serverUpdatedAt;
  if (lastSync == null) return false; // never synced: not a conflict
  return local.updatedAt.isAfter(lastSync) &&
      remote.updatedAt.isAfter(lastSync);
}

class ConflictResolver {
  ConflictResolver(this._db, this._remote);

  final AppDatabase _db;
  final PostRemote _remote;

  /// Policy 1: last write wins.
  Future<void> mergeLastWriteWins(
    PostsTableData local,
    RemotePost remote,
  ) async {
    if (local.updatedAt.isAfter(remote.updatedAt)) {
      // Local wins: push the local version to remote.
      await _remote.updatePost(local);
    } else {
      // Remote wins: overwrite local.
      await _db.postsDao.upsertPost(remote.toCompanion());
    }
  }

  /// Policy 2: server wins on protected fields.
  Future<void> applyServerAuthority(RemoteAccount account) => _db
      .into(_db.accountsTable)
      .insertOnConflictUpdate(
        AccountsTableCompanion.insert(
          id: account.id,
          displayName: account.displayName,
          subscriptionTier: account.subscriptionTier,
          accountBalance: account.accountBalance,
          syncStatus: const Value(SyncStatus.synced),
        ),
      );

  /// Policy 3: flag it. The conflict UI watches PostsDao.watchConflicted.
  Future<void> flagIfConflicted(PostsTableData local, RemotePost remote) async {
    if (isConflicted(local, remote)) {
      await _db.postsDao.upsertPost(
        local.copyWith(syncStatus: SyncStatus.conflicted),
      );
    }
  }

  /// Writes the version the person chose.
  Future<void> resolve(PostsTableData chosen, DateTime serverTime) =>
      _db.postsDao.upsertPost(
        chosen.copyWith(
          syncStatus: SyncStatus.synced,
          serverUpdatedAt: Value(serverTime),
        ),
      );
}
```

`copyWith` on a row takes the plain value, with `Value` only for nullable columns. A companion's `copyWith` takes a `Value` for every field.

## Conflict Resolution UI

Show a diff view, and let the person pick local, remote or a merge. Write the resolved version with `SyncStatus.synced`, and push it to the server through the sync queue.
