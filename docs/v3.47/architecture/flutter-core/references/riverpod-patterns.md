---
title: Riverpod provider and widget patterns
author: MOZGII, Inc.
lang: en
tags: [flutter, riverpod, hooks, state]
description: The Riverpod 3 code-generation shapes a feature uses - async providers, notifiers, ConsumerWidget and HookConsumerWidget - with loading, error and empty states.
---
# Riverpod Provider and Widget Patterns

These are the shapes a feature reaches for, all compiled against `flutter_riverpod` 3.4.3 and `riverpod_generator` 4.0.9.

## Async Data Provider With Code Generation

A functional provider with `@riverpod` is auto-disposed: it lives while something watches it. It takes the single `Ref` type that Riverpod 3 uses everywhere.

```dart
// lib/features/users/presentation/providers/user_list_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'user_list_provider.g.dart';

class AppUser {
  const AppUser(this.id, this.name);
  final String id;
  final String name;
}

abstract interface class UserRepository {
  Future<List<AppUser>> getUsers();
}

@riverpod
UserRepository userRepository(Ref ref) => throw UnimplementedError(
  'Override this in ProviderScope or bind it to GetIt',
);

@riverpod
Future<List<AppUser>> userList(Ref ref) async {
  final repo = ref.watch(userRepositoryProvider);
  return repo.getUsers();
}
```

## ConsumerWidget for a Full Widget Rebuild

`ConsumerWidget` rebuilds when a watched provider changes. Handle all three `AsyncValue` states, and give the empty list its own screen, because loading and empty look identical to a person otherwise. The retry button uses `ref.invalidate` in a callback, which is exactly where `ref.read`-style calls belong.

```dart
// lib/features/users/presentation/pages/user_list_page.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../providers/user_list_provider.dart';

class UserListPage extends ConsumerWidget {
  const UserListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final users = ref.watch(userListProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('People')),
      body: users.when(
        data: (list) => list.isEmpty
            ? const Center(child: Text('Nobody here yet'))
            : ListView.builder(
                itemCount: list.length,
                itemBuilder: (context, i) =>
                    ListTile(title: Text(list[i].name)),
              ),
        loading: () => const Center(
          child: CircularProgressIndicator(semanticsLabel: 'Loading people'),
        ),
        error: (error, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Text("Couldn't load people"),
              TextButton(
                onPressed: () => ref.invalidate(userListProvider),
                child: const Text('Try again'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## HookConsumerWidget When Hooks Are Needed

Use `HookConsumerWidget` from `hooks_riverpod` when a screen owns controllers. Hooks dispose them for you, so the widget doesn't need a `State` class.

```dart
// lib/features/users/presentation/pages/search_page.dart
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:material_ui/material_ui.dart';

class SearchPage extends HookConsumerWidget {
  const SearchPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final controller = useTextEditingController();
    return Scaffold(
      appBar: AppBar(
        title: TextField(
          controller: controller,
          decoration: const InputDecoration(hintText: 'Search people'),
        ),
      ),
    );
  }
}
```

## Reacting to Changes With ref.listen

`ref.listen` is for one-off side effects such as a snackbar or navigation. It goes in `build`, runs on change, and never rebuilds the widget. The login screen in `routing-and-login-screen.md` uses it to show a failed sign-in.
