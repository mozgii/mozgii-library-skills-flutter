---
title: Long-lived Flutter isolate worker
author: MOZGII, Inc.
lang: en
tags: [flutter, performance, isolates]
description: A typed long-lived Dart isolate for repeated background work, with error delivery and clean shutdown.
---
# Long-lived Flutter Isolate Worker

A long-lived isolate suits a stream of jobs, where spawning a new isolate for each one would waste time. The caller sends a message with its own reply port, and the worker answers on that port, so concurrent requests can't get crossed.

## JsonWorker: request, reply and shutdown

`JsonWorker.start` spawns the isolate and waits for its command port. `parse` opens one reply port per request. `dispose` sends `null`, which closes the worker's port and lets the isolate exit once its current job ends. A failure comes back as a `RemoteError` and is rethrown at the caller, so an error in the worker can't hang a pending request.

```dart
import 'dart:convert';
import 'dart:isolate';

class JsonWorker {
  JsonWorker._(this._commands);

  final SendPort _commands;

  static Future<JsonWorker> start() async {
    final ready = ReceivePort();
    await Isolate.spawn(_workerMain, ready.sendPort);
    final commands = await ready.first as SendPort; // first also closes ready
    return JsonWorker._(commands);
  }

  Future<Object?> parse(String body) async {
    final reply = ReceivePort();
    _commands.send((body, reply.sendPort));
    final result = await reply.first;
    if (result is RemoteError) throw result;
    return result;
  }

  void dispose() => _commands.send(null);

  static void _workerMain(SendPort toMain) {
    final inbox = ReceivePort();
    toMain.send(inbox.sendPort);
    inbox.listen((message) {
      if (message == null) {
        inbox.close();
        return;
      }
      final (body, replyTo) = message as (String, SendPort);
      try {
        replyTo.send(jsonDecode(body));
      } catch (error, stack) {
        replyTo.send(RemoteError('$error', '$stack'));
      }
    });
  }
}
```

## Plugins inside a background isolate

Platform channels need a binary messenger in the isolate that calls them. Send `RootIsolateToken.instance!` from the main isolate with the first message, then call `BackgroundIsolateBinaryMessenger.ensureInitialized(token)` in the worker before the first plugin call. `RootIsolateToken.instance` is `null` outside the root isolate, so read it on the main isolate.

## When a worker is the wrong tool

A job that runs once, or once in a while, is shorter with `Isolate.run`. Web has no `dart:isolate`, so an app that also ships for web uses `compute` there.
