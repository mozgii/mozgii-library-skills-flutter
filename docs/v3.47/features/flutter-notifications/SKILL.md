---
name: flutter-notifications
description: Builds push and local notifications for Flutter - FCM v1 with Firebase Installation IDs, OneSignal 5 for segmentation, flutter_local_notifications 22 with exact and inexact scheduling, the iOS permission flow, Android 13 POST_NOTIFICATIONS, a top-level background handler, the FCM registration lifecycle, and validated deep links from a tap through go_router. Use when adding push or scheduled notifications, opening a screen from a notification, handling foreground, background and terminated states, or integrating OneSignal.
metadata:
  tags:
  - flutter
  - notifications
  - fcm
  - onesignal
  - push
  - deep-linking
---
# Flutter Notifications

A notification that doesn't arrive, opens the wrong screen, or fires an hour late costs more trust than having none. This skill covers the parts that decide that: the three app states a tap can arrive in, the permission prompt you only get once, a push address that rotates behind your back, and a scheduled alarm the OS may refuse.

A notification's `route` comes from a server, so it's untrusted input. Every tap passes through a validator before reaching go_router. The background handler is always a top-level function, permissions are requested after the person sees value, Android 13+ `POST_NOTIFICATIONS` is handled, and APNs uses a `.p8` key, never a `.p12` certificate.

Use it for any push channel, local or scheduled notifications, tap to deep link navigation, OneSignal, or foreground, background and terminated handling. Skip it for in-app notification UI (`flutter-design-ui`) and real-time data updates (`flutter-database`).

## Step 0: Read Your Versions

```bash
flutter --version
flutter pub deps --style=compact | grep -E "^- (firebase_messaging|flutter_local_notifications|timezone|onesignal_flutter) "
grep -n "compileSdk\|coreLibraryDesugaring" android/app/build.gradle*
grep -rn "getInitialMessage\|\.go(.*data\[" lib          # each one is a place to check below
grep -rn "getToken\|onTokenRefresh\|deleteToken" lib     # deprecated in firebase_messaging 17
```

The pins were read from pub.dev on 2026-10-30, and the examples were compiled against them with Flutter 3.47.9 and Dart 3.13.7. The examples also import `flutter_riverpod: ^3.4.3`, `gap: ^3.0.1`, `material_ui: ^1.9.0`, `riverpod_annotation: ^4.0.7`. `firebase_messaging` 17 builds on Firebase BoM 35 and needs Android `minSdk` 24. Everything here was compiled, not run on a device, so test delivery, taps and the permission prompt on real hardware. On `package:flutter/material.dart`, swap the `material_ui` import in these examples, or run `dart fix --apply --code=migrate_design_widgets`.

```yaml
dependencies:
  firebase_core: ^5.1.0
  firebase_messaging: ^17.0.1
  flutter_local_notifications: ^22.3.3
  flutter_secure_storage: ^11.2.0
  flutter_timezone: ^5.1.1
  go_router: ^18.1.2
  timezone: ^0.11.2
```

## Choosing a Notification Service

Choose the right notification backend before writing any code.

| Scenario | Recommended service | Reason |
|---|---|---|
| Custom backend (Supabase / custom API) | **FCM directly** | Full control, no third-party SDK overhead |
| Marketing automation, segmentation, A/B | **OneSignal** | Abstracts FCM + APNs, adds analytics dashboard |
| Scheduled / local only | **flutter_local_notifications** | No remote service required |
| Enterprise with existing push infra | **FCM** | Standard, built for high volume |

If your backend sends pushes through an API call with user targeting, or the product team needs rich segmentation, use OneSignal (`references/onesignal.md`). If you need 100% control of the message payload and delivery infrastructure, use FCM directly.

**APNs setup (iOS, required for any push):** use a `.p8` Auth Key, which never expires, not a `.p12` certificate, which expires yearly. The key lives in Apple Developer under Certificates, Identifiers & Profiles, then Keys. Upload the `.p8` key to Firebase Console under Project Settings, then Cloud Messaging.

## FCM Setup: Primary Push

### Android manifest

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
  <!-- Required for Android 13+ (API 33) -->
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

  <application>
    <!-- Registers with Firebase Installation IDs. iOS needs the same in Info.plist -->
    <meta-data
      android:name="firebase_messaging_installation_id_enabled"
      android:value="true"/>
    <!-- FCM default notification channel -->
    <meta-data
      android:name="com.google.firebase.messaging.default_notification_channel_id"
      android:value="high_importance_channel"/>
    <!-- Default notification icon (white outline on colored circle) -->
    <meta-data
      android:name="com.google.firebase.messaging.default_notification_icon"
      android:resource="@drawable/ic_notification"/>
    <meta-data
      android:name="com.google.firebase.messaging.default_notification_color"
      android:resource="@color/notification_color"/>
  </application>
</manifest>
```

### Background message handler: top-level, non-negotiable

The handler must be a top-level function, not a class method or a closure, because Flutter spawns a separate isolate for it. Passing anything else throws `ArgumentError`. It runs with no UI, so it can only handle data and has to initialise Firebase itself.

```dart
// lib/core/notifications/fcm_handler.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/foundation.dart';

import '../../firebase_options.dart';

@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  // Handle data-only background messages here.
  // Do NOT update UI: this runs in a background isolate.
  debugPrint('Background message: ${message.messageId}');
}
```

## Worked Example: Every Tap Reaches a Validated Route

A tap arrives in three ways: `onMessageOpenedApp` from the background, `getInitialMessage` from terminated, and a local notification's payload. All three must go through one path, and the router may not exist yet when the first arrives.

### Validating a notification route

Push payloads are written by servers, consoles and sometimes marketing tools, so `route` is untrusted. An unvalidated `router.go(route)` lets a compromised sender open any screen with any parameter. Accept a fixed set of prefixes and a strict character set, and reject the rest silently.

```dart
// lib/core/notifications/safe_route.dart
const _allowedPrefixes = ['/orders/', '/chat/', '/profile/'];
final _safePath = RegExp(r'^/[A-Za-z0-9_-]+(/[A-Za-z0-9_-]+)*$');

/// Returns [raw] only if it is an allowed in-app path.
String? safeNotificationRoute(String? raw) {
  if (raw == null || !_safePath.hasMatch(raw)) return null;
  return _allowedPrefixes.any(raw.startsWith) ? raw : null;
}
```

### NotificationService: one path for every tap

`initialize()` runs in `main()` before `runApp()`, and it registers the background handler first. It doesn't touch `getInitialMessage`. A tap that lands before the router exists is held in `_pendingRoute`.

`handleInitialMessage` is the only caller of `getInitialMessage`, and it runs once, after the router is attached. The call consumes the message: a second one returns `null`, so a duplicate check elsewhere silently loses the deep link.

```dart
// lib/core/notifications/notification_service.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:go_router/go_router.dart';

import 'fcm_handler.dart';
import 'safe_route.dart';

class NotificationService {
  NotificationService(this._messaging, this._localNotifications);

  final FirebaseMessaging _messaging;
  final FlutterLocalNotificationsPlugin _localNotifications;

  GoRouter? _router;
  String? _pendingRoute;
  bool _initialMessageHandled = false;

  static const _channelId = 'high_importance_channel';
  static const _channelName = 'High Importance Notifications';

  Future<void> initialize() async {
    // Register the background handler before anything else.
    FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);

    await _localNotifications.initialize(
      settings: const InitializationSettings(
        android: AndroidInitializationSettings('@drawable/ic_notification'),
        iOS: DarwinInitializationSettings(
          requestAlertPermission: false, // Managed separately
          requestBadgePermission: false,
          requestSoundPermission: false,
        ),
      ),
      onDidReceiveNotificationResponse: (details) => _open(details.payload),
    );

    // Create the Android notification channel (required for Android 8+).
    await _localNotifications
        .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin
        >()
        ?.createNotificationChannel(
          const AndroidNotificationChannel(
            _channelId,
            _channelName,
            importance: Importance.max,
          ),
        );

    // FCM does NOT show a heads-up notification in the foreground on Android.
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // Handle a tap while the app is in the background (not terminated).
    FirebaseMessaging.onMessageOpenedApp.listen(
      (message) => _open(message.data['route'] as String?),
    );
  }

  /// Call once, after the router exists.
  Future<void> attachRouter(GoRouter router) async {
    _router = router;
    final pending = _pendingRoute;
    _pendingRoute = null;
    if (pending != null) router.go(pending);
    await handleInitialMessage();
  }

  /// Handles the tap that launched the app from the terminated state.
  Future<void> handleInitialMessage() async {
    if (_initialMessageHandled) return;
    _initialMessageHandled = true;

    final message = await _messaging.getInitialMessage();
    if (message != null) {
      _open(message.data['route'] as String?);
      return;
    }
    final launch = await _localNotifications.getNotificationAppLaunchDetails();
    if (launch?.didNotificationLaunchApp ?? false) {
      _open(launch?.notificationResponse?.payload);
    }
  }

  void _open(String? rawRoute) {
    final route = safeNotificationRoute(rawRoute);
    if (route == null) return;
    final router = _router;
    if (router == null) {
      _pendingRoute = route;
    } else {
      router.go(route);
    }
  }

  void _handleForegroundMessage(RemoteMessage message) {
    final notification = message.notification;
    if (notification == null) return;

    _localNotifications.show(
      id: (message.messageId ?? notification.title).hashCode & 0x7fffffff,
      title: notification.title,
      body: notification.body,
      notificationDetails: const NotificationDetails(
        android: AndroidNotificationDetails(
          _channelId,
          _channelName,
          importance: Importance.max,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      payload: message.data['route'] as String?, // Deep link route
    );
  }
}
```

### Deep Link Navigation on Tap

Attach the router after it's mounted, in the first frame. Attaching there avoids a circular dependency between the service and the router. A payload uses the format `/feature/resourceId`, such as `/orders/order-123`.

```dart
// lib/app_root.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';

import 'core/notifications/notification_service.dart';

final routerProvider = Provider<GoRouter>(
  (ref) => throw UnimplementedError('Bind this to your router'),
);
final notificationServiceProvider = Provider<NotificationService>(
  (ref) => throw UnimplementedError('Bind this in main()'),
);

class AppRoot extends ConsumerStatefulWidget {
  const AppRoot({super.key});

  @override
  ConsumerState<AppRoot> createState() => _AppRootState();
}

class _AppRootState extends ConsumerState<AppRoot> {
  @override
  void initState() {
    super.initState();
    // Wire notification routing after the router is mounted.
    WidgetsBinding.instance.addPostFrameCallback((_) {
      ref
          .read(notificationServiceProvider)
          .attachRouter(ref.read(routerProvider));
    });
  }

  @override
  Widget build(BuildContext context) =>
      MaterialApp.router(routerConfig: ref.watch(routerProvider));
}
```

## Permissions, Tokens and Scheduling

| Topic | Rule | Reference |
|---|---|---|
| iOS permission flow | Never on first launch. Show an explainer, then the system prompt after the person takes an action that implies value | `references/permissions.md` |
| FCM registration lifecycle | `getToken` and `onTokenRefresh` are deprecated. Subscribe to `onRegistered` before any early return, upload only on change, and call `register()` only after a permission grant, because it can prompt | `references/token-lifecycle.md` |
| Local and scheduled notifications | Initialise `tz`, and fall back to inexact when the exact alarm permission is off | `references/local-notifications.md` |
| OneSignal | Initialise once, log in after identity is established, and ask for permission in context | `references/onesignal.md` |

`flutter_local_notifications` 20 made every parameter of `initialize`, `show`, `zonedSchedule` and `cancel` a named one, and renamed `details` to `notificationDetails`. Version 19 removed `uiLocalNotificationDateInterpretation`, and 18 removed `androidAllowWhileIdle` and made `androidScheduleMode` required. Version 21 needs Flutter 3.38.1 and `compileSdk` 36, and its readme covers the extra steps for apps on the `UIScene` lifecycle.

## Verify

The route validator is a security check, so its attack cases are a test. Imports use the package name `app`: replace it with yours.

```dart
// test/core/notifications/safe_route_test.dart
import 'package:app/core/notifications/safe_route.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  test('allowed in-app paths pass', () {
    expect(safeNotificationRoute('/orders/order-123'), '/orders/order-123');
    expect(safeNotificationRoute('/chat/room_1/thread'), '/chat/room_1/thread');
  });

  test('attacks and unknown routes are refused', () {
    for (final attack in [
      null,
      '',
      '//evil.example',
      '/\t/evil.example',
      '/\n/evil.example',
      'https://evil.example',
      'javascript:alert(1)',
      '/orders/../admin',
      '/orders/1?next=//evil.example',
      '/orders/1\n',
      '/orders/',
      '/admin/users',
    ]) {
      expect(safeNotificationRoute(attack), isNull, reason: '$attack');
    }
  });
}
```

Then run `flutter analyze` and `flutter test`, and on a real device (FCM doesn't reach an iOS simulator reliably), send one notification with a `route` in each state:

1. **Foreground:** a heads-up appears, and tapping it opens the route.
2. **Background:** tapping opens the route.
3. **Terminated:** kill the app, and tapping opens the route exactly once.
4. **Bad route:** send `//evil.example`, and the app opens normally on its home screen.
5. **Scheduled:** turn off the exact alarm permission on Android 14, and the reminder still fires, a little late.

## Gotchas

1. **The app opens on home instead of the deep link after a cold start.** Symptom: terminated taps do nothing. Cause: `getInitialMessage` was called twice, and the first call consumed the message before the router was ready. Fix: call it once, in `attachRouter`.
2. **A tap during startup does nothing.** Symptom: the app opens, and the deep link is lost with no error. Cause: the tap arrived before the router existed. Fix: hold it in `_pendingRoute`.
3. **Pushes vanish after an address rotates.** Symptom: the server sends to a stale address and nothing errors. Cause: the listener came after an early return for an unchanged value. Fix: subscribe to `onRegistered` first.
4. **`apns-token-not-set` on iOS.** Symptom: an FCM call throws on the first launch. Cause: APNs hasn't delivered a token to the device yet. Fix: wait for `getAPNSToken()`, as in `references/token-lifecycle.md`.
5. **`LocationNotFoundException` or `tz.local` shows UTC.** Symptom: a crash or a reminder hours off. Cause: the timezone data or local location wasn't set. Fix: run `tz.initializeTimeZones()` and `tz.setLocalLocation`.
6. **`exact_alarms_not_permitted`.** Symptom: scheduling throws on Android 14. Cause: on Android 14 and later, a fresh install usually starts with this permission denied. Fix: check `canScheduleExactNotifications()` and fall back to inexact.
7. **`Invalid argument (id): must fit within the size of a 32-bit integer`.** Symptom: `show` throws on some messages. Cause: a raw hash as the id. Fix: mask it with `0x7fffffff`.
8. **The same banner appears twice on iOS.** Symptom: one push, two notifications. Cause: `setForegroundNotificationPresentationOptions` plus a local `show`. Fix: use one of them.
9. **The permission prompt appears at launch.** Symptom: the system dialog shows before any explainer. Cause: `register()` asks when permission isn't granted. Fix: rely on auto-init until a grant, then call it.
10. **A `.p12` push stops working a year later.** Symptom: iOS pushes fail suddenly. Cause: the certificate expired. Fix: switch to a `.p8` key.

## Anti-Patterns

| Don't | Do instead |
|---|---|
| `router.go(message.data['route'])` | `safeNotificationRoute` first |
| Prompt for permission on first launch | An explainer after a moment of value |
| A background handler inside a class | A top-level function with `@pragma('vm:entry-point')` |
| Update UI from the background handler | Store the data, and let the app read it on resume |
| Require exact alarms for a reminder | Fall back to `inexactAllowWhileIdle` |
| Mix `getToken` with `register()` | The FID APIs only, with the opt-in flags |
| Send secrets in a push payload | Send an id, and fetch over your API |

## Pre-Merge Checklist

- [ ] The attack test for `safeNotificationRoute` passes
- [ ] `getInitialMessage` has exactly one caller
- [ ] All tap sources go through `_open`
- [ ] `onRegistered` is subscribed before any early return, and cancelled on sign-out
- [ ] The FID opt-in flags are set on Android and iOS, and `register()` waits for a permission grant
- [ ] `tz` is initialised before the first schedule
- [ ] Exact scheduling has an inexact fallback
- [ ] Each of the three states was tested on a real device
- [ ] iOS uses a `.p8` key

## See Also

`flutter-setup-declarative-routing` (go_router), `flutter-core` (architecture), `flutter-security` (payload handling), `flutter-analytics-firebase` (measuring opens), `flutter-state` (app lifecycle).

Depth in this skill: `references/permissions.md`, `references/token-lifecycle.md`, `references/local-notifications.md`, `references/onesignal.md`.
