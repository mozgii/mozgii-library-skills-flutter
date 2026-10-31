---
title: The iOS notification permission flow
author: MOZGII, Inc.
lang: en
tags: [flutter, notifications, permissions, ios]
description: Asking for notification permission at the right moment, with a Riverpod provider and an explainer sheet that raises opt-in.
---
# The iOS Notification Permission Flow

iOS shows its permission prompt once. A refusal can only be undone in Settings, so the moment you ask decides whether notifications are ever received at all.

## When to Ask for Notification Permission

Never request permission on first launch. Show the request after the person takes an action that implies value, such as enabling a feature or completing onboarding. A pre-permission explainer of your own comes first: if they decline that, the system prompt is still unused.

## Permission Provider With Riverpod

`request()` covers Android 13+ as well, since `requestPermission()` shows the `POST_NOTIFICATIONS` prompt there. `provisional: false` means the full permission dialog.

```dart
// lib/features/notifications/presentation/providers/permission_provider.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'permission_provider.g.dart';

@riverpod
class NotificationPermission extends _$NotificationPermission {
  @override
  Future<AuthorizationStatus> build() async {
    final settings = await FirebaseMessaging.instance.getNotificationSettings();
    return settings.authorizationStatus;
  }

  /// Call this only after showing the person why they need notifications.
  Future<void> request() async {
    final settings = await FirebaseMessaging.instance.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false, // false = full permission dialog
    );
    state = AsyncData(settings.authorizationStatus);
  }
}
```

## Pre-Permission Explainer Sheet

Read the notifier before `Navigator.pop`. Popping disposes this widget, and using `ref` after that throws.

```dart
// lib/features/notifications/presentation/widgets/notification_permission_sheet.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:gap/gap.dart';
import 'package:material_ui/material_ui.dart';

import '../providers/permission_provider.dart';

class NotificationPermissionSheet extends ConsumerWidget {
  const NotificationPermissionSheet({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Icon(Icons.notifications_outlined, size: 64),
          Text(
            'Stay in the loop',
            style: Theme.of(context).textTheme.headlineSmall,
          ),
          const Gap(8),
          const Text(
            'Get notified when your order ships, your friend replies, or '
            'something needs your attention.',
            textAlign: TextAlign.center,
          ),
          const Gap(24),
          FilledButton(
            onPressed: () {
              final permission = ref.read(
                notificationPermissionProvider.notifier,
              );
              Navigator.pop(context);
              permission.request();
            },
            child: const Text('Enable Notifications'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Not now'),
          ),
        ],
      ),
    );
  }
}
```

When the status is `AuthorizationStatus.denied`, the prompt won't appear again, so offer a button to the system settings instead. `flutter_local_notifications` 22.3 has `openAppNotificationSettings()` for that.
