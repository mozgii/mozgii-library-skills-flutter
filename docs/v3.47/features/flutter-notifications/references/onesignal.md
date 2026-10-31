---
title: OneSignal for segmentation and marketing
author: MOZGII, Inc.
lang: en
tags: [flutter, onesignal, push, marketing]
description: When OneSignal beats FCM directly, and a thin service that initialises it, ties it to a signed-in user, and tags for segments.
---
# OneSignal for Segmentation and Marketing

Use OneSignal when the product team needs rich segmentation, A/B testing, delivery analytics, in-app messages, or multi-channel (push, email and SMS) from one dashboard.

```yaml
dependencies:
  onesignal_flutter: ^5.7.2
```

## OneSignalService: Initialise, Log In, Tag

Initialise once at startup, without requesting permission. Ask in context, as `permissions.md` describes, by calling `OneSignal.Notifications.requestPermission(true)` yourself. Call `OneSignal.login` only after user identity is established, and `logout` on sign-out, so the device stops getting that person's segments.

```dart
// lib/core/notifications/onesignal_service.dart
import 'package:onesignal_flutter/onesignal_flutter.dart';

class OneSignalService {
  Future<void> initialize(String appId) async {
    // Do NOT request permission here: ask contextually.
    await OneSignal.initialize(appId);
  }

  /// Call after user identity is established (post-login).
  Future<void> setExternalUserId(String userId) => OneSignal.login(userId);

  /// Tag the user for segmentation.
  Future<void> setUserTag(String key, String value) =>
      OneSignal.User.addTagWithKey(key, value);

  /// Call from your pre-permission explainer.
  Future<bool> requestPermission() =>
      OneSignal.Notifications.requestPermission(true);

  Future<void> logout() => OneSignal.logout();
}
```

Anything OneSignal delivers can carry a deep link too. Pass its route through the same validator as an FCM message.
