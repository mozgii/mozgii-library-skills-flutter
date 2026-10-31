---
title: Reacting to connectivity in the UI
author: MOZGII, Inc.
lang: en
tags: [flutter, connectivity, offline, riverpod]
description: Refetching data when the network returns and an offline banner that screen readers announce only when it matters.
---
# Reacting to Connectivity in the UI

These two pieces sit on top of `ConnectivityNotifier` in the skill. The first refetches data when the network returns, and the second tells the person they're offline.

## Invalidate Providers on Network Restore

A data provider can auto-refetch when connectivity restores by listening to the connectivity provider inside `build`. `ref.invalidateSelf()` runs only on the offline-to-online edge, so a change from Wi-Fi to mobile data doesn't reload anything.

```dart
// lib/features/posts/presentation/providers/posts_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/state/connectivity_notifier.dart';

part 'posts_provider.g.dart';

class Post {
  const Post(this.id, this.title);
  final String id;
  final String title;
}

abstract interface class PostRepository {
  Future<List<Post>> getPosts();
}

@riverpod
PostRepository postRepository(Ref ref) =>
    throw UnimplementedError('Bind this to your repository');

@riverpod
class PostsNotifier extends _$PostsNotifier {
  @override
  Future<List<Post>> build() async {
    ref.listen(connectivityProvider, (prev, next) {
      final wasOffline = prev is ConnectivityStateOffline;
      final nowOnline = next is ConnectivityStateOnline;
      if (wasOffline && nowOnline) ref.invalidateSelf();
    });
    return ref.watch(postRepositoryProvider).getPosts();
  }
}
```

## Offline Banner Widget

The banner slides in from the top. While it's off screen it's still in the tree, so `ExcludeSemantics` stops a screen reader from reading "No internet connection" to someone who's online, and `liveRegion` announces it the moment it appears.

```dart
// lib/shared/widgets/offline_banner.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../../core/state/connectivity_notifier.dart';

class OfflineBanner extends ConsumerWidget {
  const OfflineBanner({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final connectivity = ref.watch(connectivityProvider);
    final isOffline = connectivity is ConnectivityStateOffline;
    final colors = Theme.of(context).colorScheme;
    return ExcludeSemantics(
      excluding: !isOffline,
      child: Semantics(
        liveRegion: true,
        child: AnimatedSlide(
          offset: isOffline ? Offset.zero : const Offset(0, -1),
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeInOut,
          child: Container(
            width: double.infinity,
            color: colors.error,
            padding: const EdgeInsets.symmetric(vertical: 6, horizontal: 16),
            child: Text(
              'No internet connection',
              style: Theme.of(context).textTheme.labelMedium
                  ?.copyWith(color: colors.onError),
              textAlign: TextAlign.center,
            ),
          ),
        ),
      ),
    );
  }
}
```
