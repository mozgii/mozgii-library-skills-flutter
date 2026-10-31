---
title: State hydration on cold start
author: MOZGII, Inc.
lang: en
tags: [flutter, hive, startup, hydration, riverpod]
description: One gate provider that opens Hive boxes and restores the cached session before the app appears, with a recovery path for corrupt cache and a retry on failure.
---
# State Hydration on Cold Start

Cold start restores what the app knew when it closed. One gate provider does all of it, and every other startup read waits for that gate.

## The Startup Hydration Provider

`startupHydrationProvider` makes sure all Hive boxes are open, then restores the cached user into `currentUserProvider`. It's `keepAlive`, so it runs once per launch. A cached value that no longer parses is deleted and treated as signed out: a bad cache entry must never stop the app from starting.

```dart
// lib/core/state/startup_provider.dart
import 'dart:convert';

import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'startup_provider.g.dart';

class CachedUser {
  const CachedUser({required this.id, required this.name});

  factory CachedUser.fromJson(Map<String, dynamic> json) =>
      CachedUser(id: json['id'] as String, name: json['name'] as String);

  final String id;
  final String name;
}

@Riverpod(keepAlive: true)
class CurrentUserNotifier extends _$CurrentUserNotifier {
  @override
  CachedUser? build() => null;

  void setUser(CachedUser? user) => state = user;
}

/// Single gate provider: all other startup state reads wait for this.
@Riverpod(keepAlive: true)
Future<void> startupHydration(Ref ref) async {
  // Ensure all Hive boxes are open before any state reads.
  await Future.wait([
    Hive.openBox<String>('settings'),
    Hive.openBox<String>('user_cache'),
    Hive.openBox<Map<dynamic, dynamic>>('offline_queue'),
  ]);

  // Hydrate the cached user session.
  final userBox = Hive.box<String>('user_cache');
  final cachedUser = userBox.get('current_user');
  if (cachedUser == null) return;
  try {
    final json = jsonDecode(cachedUser) as Map<String, dynamic>;
    ref.read(currentUserProvider.notifier).setUser(CachedUser.fromJson(json));
  } on FormatException {
    await userBox.delete('current_user');
  } on TypeError {
    await userBox.delete('current_user');
  }
}
```

## Gating the First Frame on Hydration

`StartupGate` shows a splash until the gate finishes. Both of its `MaterialApp`s take the saved `themeMode` and a dark `darkTheme`: `main()` has already opened the settings box, and without them a dark-mode person sees a light frame first. If hydration fails, the person gets a message and a retry through `ref.invalidate`, not a splash screen that never ends. Boxes usually fail because a device is out of storage, and that can clear up.

```dart
// lib/core/state/startup_gate.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../app.dart';
import 'startup_provider.dart';
import 'theme_notifier.dart';

class StartupGate extends ConsumerWidget {
  const StartupGate({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final hydration = ref.watch(startupHydrationProvider);
    final themeMode = ref.watch(themeModeProvider);

    Widget screen(Widget child) => MaterialApp(
      theme: ThemeData(),
      darkTheme: ThemeData(brightness: Brightness.dark),
      themeMode: themeMode,
      home: Scaffold(body: Center(child: child)),
    );

    return hydration.when(
      loading: () => screen(
        const CircularProgressIndicator(semanticsLabel: 'Starting'),
      ),
      error: (_, _) => screen(
        Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text("Couldn't open your data"),
            TextButton(
              onPressed: () => ref.invalidate(startupHydrationProvider),
              child: const Text('Try again'),
            ),
          ],
        ),
      ),
      data: (_) => const MainApp(),
    );
  }
}
```

## Hydration Test: A Corrupt Cache Can't Stop Startup

This test writes a broken value into a real Hive box in a temporary folder and proves the gate still completes.

```dart
// test/core/state/startup_provider_test.dart
import 'dart:io';

import 'package:app/core/state/startup_provider.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';

void main() {
  late Directory dir;

  setUp(() async {
    dir = await Directory.systemTemp.createTemp('hydration');
    Hive.init(dir.path);
  });

  tearDown(() async {
    await Hive.close();
    await dir.delete(recursive: true);
  });

  Future<CachedUser?> hydrateWith(String stored) async {
    final box = await Hive.openBox<String>('user_cache');
    await box.put('current_user', stored);
    final container = ProviderContainer();
    addTearDown(container.dispose);
    await container.read(startupHydrationProvider.future);
    return container.read(currentUserProvider);
  }

  test('a valid cache restores the user', () async {
    final user = await hydrateWith('{"id":"1","name":"Ada"}');
    expect(user?.name, 'Ada');
  });

  test('corrupt JSON is removed and startup completes', () async {
    expect(await hydrateWith('{not json'), isNull);
    expect(Hive.box<String>('user_cache').get('current_user'), isNull);
  });

  test('a wrong shape is removed and startup completes', () async {
    expect(await hydrateWith('{"id":1}'), isNull);
  });
}
```
