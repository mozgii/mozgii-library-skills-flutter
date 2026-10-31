---
title: Local and scheduled notifications
author: MOZGII, Inc.
lang: en
tags: [flutter, notifications, scheduling, timezone, android]
description: Scheduling reminders with flutter_local_notifications 22, timezone setup, Android exact alarm permission with an inexact fallback, and the manifest and Gradle steps they need.
---
# Local and Scheduled Notifications

Local notifications need no server, but a reminder depends on three things the OS controls: the time zone, an alarm permission and a reboot.

```yaml
dependencies:
  flutter_local_notifications: ^22.3.3
  flutter_timezone: ^5.1.1
  timezone: ^0.11.2
```

## Android Setup for Scheduled Notifications

Since version 16, the plugin declares only `POST_NOTIFICATIONS` and `VIBRATE`. Scheduling needs more in your own `AndroidManifest.xml`. `RECEIVE_BOOT_COMPLETED` and the boot receiver let the plugin reschedule after a reboot.

```xml
<manifest>
  <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
  <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>

  <application>
    <receiver android:exported="false"
      android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver"/>
    <receiver android:exported="false"
      android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
      <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
        <action android:name="android.intent.action.QUICKBOOT_POWERON"/>
        <action android:name="com.htc.intent.action.QUICKBOOT_POWERON"/>
      </intent-filter>
    </receiver>
  </application>
</manifest>
```

The plugin relies on core library desugaring, even if you don't schedule anything, and on `compileSdk` 36:

```kotlin
// android/app/build.gradle.kts
android {
    compileSdk = 36
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
    }
}
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4")
}
```

## Exact Alarms and the Inexact Fallback

On Android 14 and later, a fresh install of an app targeting API 33 or higher usually starts with `SCHEDULE_EXACT_ALARM` denied. In that state, `AndroidScheduleMode.exactAllowWhileIdle` fails with `exact_alarms_not_permitted`. There are two ways to get exact timing:

| Permission | Prompt | Use when |
|---|---|---|
| `SCHEDULE_EXACT_ALARM` + `requestExactAlarmsPermission()` | The person grants it in system settings | Exact timing helps but isn't the app's purpose |
| `USE_EXACT_ALARM` | None, but it needs API 33+ and you can't publish with it unless your app qualifies under store policy | An alarm clock or calendar app |

Check before each schedule rather than once, because the person can revoke the permission at any time. For a reminder, `inexactAllowWhileIdle` is the right fallback: a few minutes late beats not firing.

## Timezone Setup for zonedSchedule

`zonedSchedule` takes a `TZDateTime`, so `timezone` needs its database and the device's location. The `timezone` package can't read the device's zone itself, and `DateTime.now().timeZoneName` gives an abbreviation rather than an IANA name. `flutter_timezone` provides the identifier, and without this setup `tz.local` is UTC and every reminder is off by the person's offset.

```dart
// lib/core/notifications/local_notification_service.dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:flutter_timezone/flutter_timezone.dart';
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

class LocalNotificationService {
  LocalNotificationService(this._plugin);
  final FlutterLocalNotificationsPlugin _plugin;

  AndroidFlutterLocalNotificationsPlugin? get _android => _plugin
      .resolvePlatformSpecificImplementation<
        AndroidFlutterLocalNotificationsPlugin
      >();

  Future<void> initialize(void Function(String?) onTap) async {
    tz.initializeTimeZones();
    final local = await FlutterTimezone.getLocalTimezone();
    tz.setLocalLocation(tz.getLocation(local.identifier));

    await _plugin.initialize(
      settings: const InitializationSettings(
        android: AndroidInitializationSettings('@drawable/ic_notification'),
        iOS: DarwinInitializationSettings(
          requestAlertPermission: false, // Managed separately
          requestBadgePermission: false,
          requestSoundPermission: false,
        ),
      ),
      onDidReceiveNotificationResponse: (details) => onTap(details.payload),
    );
  }

  /// Opens the system page where the person can allow exact alarms.
  Future<bool> requestExactAlarms() async =>
      await _android?.requestExactAlarmsPermission() ?? false;

  Future<AndroidScheduleMode> _scheduleMode() async {
    final android = _android;
    if (android == null) return AndroidScheduleMode.exactAllowWhileIdle;
    final allowed = await android.canScheduleExactNotifications() ?? false;
    return allowed
        ? AndroidScheduleMode.exactAllowWhileIdle
        : AndroidScheduleMode.inexactAllowWhileIdle;
  }

  /// Schedules a notification, exact when the OS allows it.
  Future<void> schedule({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledAt,
    String? payload,
  }) async {
    await _plugin.zonedSchedule(
      id: id,
      title: title,
      body: body,
      scheduledDate: tz.TZDateTime.from(scheduledAt, tz.local),
      notificationDetails: const NotificationDetails(
        android: AndroidNotificationDetails(
          'scheduled_channel',
          'Scheduled Reminders',
          importance: Importance.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      payload: payload,
      androidScheduleMode: await _scheduleMode(),
    );
  }

  Future<void> cancel(int id) => _plugin.cancel(id: id);
  Future<void> cancelAll() => _plugin.cancelAll();
}
```

A `payload` from `schedule` comes back through `onTap` and must pass the same route validator as a push. A local payload is written by your own code, but it's stored on the device.
