---
title: The sync queue service
author: MOZGII, Inc.
lang: en
tags: [flutter, sync, offline-first, drift, connectivity]
description: SyncService drains a Drift-backed queue in order, one drain at a time, retries after a reconnect and parks writes the server will never accept.
---
# The Sync Queue Service

`SyncService` drains the queue stored in Drift. It compares `onConnectivityChanged` results with `contains`, drains once at start, stops at the first network failure to keep order, and reads the queue again until it's empty, so a write made during a drain goes out in that drain. `_send` throws `PermanentSyncFailure` for a write the server will never accept, which parks it.

```dart
// lib/core/sync/sync_service.dart
import 'dart:async';

import 'package:connectivity_plus/connectivity_plus.dart';

import '../database/app_database.dart';

typedef SendOperation = Future<void> Function(SyncOperationsTableData op);

/// Thrown by a [SendOperation] when retrying can never succeed.
class PermanentSyncFailure implements Exception {
  const PermanentSyncFailure(this.reason);
  final String reason;
}

class SyncService {
  SyncService(this._dao, this._connectivity, this._send);

  final SyncQueueDao _dao;
  final Connectivity _connectivity;
  final SendOperation _send;
  StreamSubscription<List<ConnectivityResult>>? _sub;
  bool _draining = false;

  void start() {
    _sub ??= _connectivity.onConnectivityChanged.listen((results) {
      if (!results.contains(ConnectivityResult.none)) unawaited(drainQueue());
    });
    unawaited(drainQueue()); // writes left from the last run
  }

  Future<void> stop() async {
    await _sub?.cancel();
    _sub = null;
  }

  Future<void> drainQueue() async {
    if (_draining) return;
    _draining = true;
    try {
      while (true) {
        final pending = await _dao.getPending(); // includes newer writes
        if (pending.isEmpty) return;
        for (final op in pending) {
          try {
            await _send(op);
            await _dao.markComplete(op.id);
          } on PermanentSyncFailure {
            await _dao.park(op.id, op.attempts + 1);
          } on Exception {
            await _dao.recordFailure(op.id, op.attempts + 1);
            return; // keep order: retry from here on the next restore
          }
        }
      }
    } finally {
      _draining = false;
    }
  }
}
```
