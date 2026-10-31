---
title: An offline write queue
author: MOZGII, Inc.
lang: en
tags: [flutter, offline, connectivity, sync]
description: Keeping writes made offline in durable storage and replaying them once, in order, when the network returns.
---
# An Offline Write Queue

An offline queue saves a write the network can't take yet and sends it later. The rule that makes it safe: a queued write may be sent twice, for example when the app dies after the server replied. So each one carries an `Idempotency-Key` created when the person acted, and your server has to honour it.

```yaml
dependencies:
  connectivity_plus: ^7.3.1    # below 7.3.2: its dbus 0.8 conflicts with geolocator
```

## OfflineQueue Over Any Durable Store

The store is an interface, so the queue works over Drift, Hive or any other durable storage. `flutter-database` has a Drift table that fits it. `start()` drains once, because connectivity events only fire on a change, and an app that opens online with writes left from its last run would otherwise never send them. `drainQueue` runs one drain at a time, so a write saved during a drain can't start its own, and the running drain reads the store again until nothing is pending. It keeps order and stops at the first failure that can pass with time, because the rest will fail too: no response, a 5xx, 401 (the session needs a refresh), 408 (timeout), 425 (too early) and 429 (rate limited). Any other 4xx will never succeed, so that write is parked instead of blocking everything behind it.

```dart
// lib/core/network/offline_queue.dart
import 'dart:async';

import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:dio/dio.dart';

class QueuedRequest {
  const QueuedRequest({
    required this.id,
    required this.method,
    required this.path,
    required this.body,
    required this.idempotencyKey,
  });

  final int id;
  final String method;
  final String path;
  final Map<String, Object?> body;
  final String idempotencyKey;
}

abstract interface class QueueStore {
  Future<void> insert({
    required String method,
    required String path,
    required Map<String, Object?> body,
    required String idempotencyKey,
  });

  /// Leaves out completed and failed requests, or a drain would never end.
  Future<List<QueuedRequest>> pendingOldestFirst();
  Future<void> markCompleted(int id);
  Future<void> markFailed(int id, String reason);
}

class OfflineQueue {
  OfflineQueue(this._store, this._connectivity, this._dio);

  /// 4xx answers that can pass with time, so the write stays queued.
  static const _retryLater = {401, 408, 425, 429};

  final QueueStore _store;
  final Connectivity _connectivity;
  final Dio _dio;
  StreamSubscription<List<ConnectivityResult>>? _sub;
  bool _draining = false;

  /// Sends what the last run left, then drains when connectivity returns.
  void start() {
    _sub ??= _connectivity.onConnectivityChanged.listen((results) {
      if (!results.contains(ConnectivityResult.none)) unawaited(drainQueue());
    });
    unawaited(drainQueue());
  }

  Future<void> dispose() async => _sub?.cancel();

  /// Saves a write. It goes out at once if the device is online.
  Future<void> enqueue({
    required String method,
    required String path,
    required Map<String, Object?> body,
    required String idempotencyKey,
  }) async {
    await _store.insert(
      method: method,
      path: path,
      body: body,
      idempotencyKey: idempotencyKey,
    );
    final state = await _connectivity.checkConnectivity();
    if (!state.contains(ConnectivityResult.none)) await drainQueue();
  }

  Future<void> drainQueue() async {
    if (_draining) return;
    _draining = true;
    try {
      while (true) {
        final pending = await _store.pendingOldestFirst();
        if (pending.isEmpty) return;
        for (final request in pending) {
          try {
            await _execute(request);
            await _store.markCompleted(request.id);
          } on DioException catch (e) {
            final status = e.response?.statusCode;
            if (status == null || status >= 500) return; // try again later
            if (_retryLater.contains(status)) return;
            await _store.markFailed(request.id, 'HTTP $status');
          }
        }
      }
    } finally {
      _draining = false;
    }
  }

  Future<void> _execute(QueuedRequest request) => _dio.request<void>(
    request.path,
    data: request.body,
    options: Options(
      method: request.method,
      headers: {'Idempotency-Key': request.idempotencyKey},
    ),
  );
}
```

The key is what lets `RetryInterceptor` retry these `POST` requests as well. Tell the person when a write ends up in `markFailed`, since it won't be sent again and they may have to redo it.
