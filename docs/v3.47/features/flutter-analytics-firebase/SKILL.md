---
name: flutter-analytics-firebase
description: Sets up Firebase Analytics, Crashlytics and Performance Monitoring in a Flutter app with consent first - collection off by default in native config, a typed event service, go_router screen tracking, non-fatal reporting, HTTP traces and a GDPR consent gate. Use when adding analytics, crash reporting or performance monitoring, defining events, tracking screens, or when a privacy review asks what is collected before consent.
metadata:
  tags:
  - flutter
  - firebase
  - analytics
  - crashlytics
  - performance
  - gdpr
---
# Flutter Analytics (Firebase)

Measurement for a production Flutter app that doesn't collect anything it shouldn't. All three Firebase products collect by default from the first launch, so a consent screen built only in Dart runs too late. This skill turns collection off in native config, turns it on after consent, and keeps personal data out of every event.

Skip it if the project uses another analytics provider. Adding one event to an existing typed service needs only a new method.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(firebase_core|firebase_analytics|firebase_crashlytics|firebase_performance|crypto|hive_ce_flutter|go_router):" pubspec.yaml
dart pub outdated --no-dev-dependencies
```

Pins were read from pub.dev on 2026-10-30 for Flutter 3.47.9 and Dart 3.13.7, and the examples were analyzed against them. `build_runner: ^2.16.1` sits below the latest 2.17.1 on purpose: 2.17.1 needs `build` 5, which `envied_generator`, `hive_ce_generator`, `injectable_generator`, `isar_community_generator` and `riverpod_generator` don't accept yet, and 2.16.2 needs `analyzer` 14.3, which `injectable_generator` 3.1.3 can't take. On `package:flutter/material.dart`, swap the `material_ui` import in these examples, or run `dart fix --apply --code=migrate_design_widgets`. Everything here was compiled, not run on a device, so confirm your events in DebugView from a real build.

```yaml
dependencies:
  firebase_core: ^5.1.0
  firebase_analytics: ^13.0.1
  firebase_crashlytics: ^6.0.1
  firebase_performance: ^0.12.1
  crypto: ^3.0.8
  flutter_riverpod: ^3.4.3
  riverpod_annotation: ^4.0.7
  hive_ce_flutter: ^2.5.0
```

The examples also import `dio: ^5.11.1`, `go_router: ^18.1.2`, `material_ui: ^1.9.0`.

## Choosing What to Measure

| Question | Use | Why the others lose here |
|---|---|---|
| What do users do? | Analytics events | Crash keys aren't queryable as funnels |
| What broke, and where? | Crashlytics | Analytics events carry no stack trace |
| A caught error worth tracking | `recordError(fatal: false)` | Analytics can't group by stack |
| Expected failures (wrong password, no results) | Nothing, or one event | Non-fatals would bury real bugs; Crashlytics stores only the latest eight |
| How slow is it? | Performance traces | Timestamps in events can't give percentiles |

## Collection Off by Default in Native Config

Analytics, Crashlytics and Performance Monitoring all collect by default, and they start before your first Dart frame. Set these keys first. Each runtime call below overrides them and persists across launches, so a user's consent stays in effect after you ship this.

```xml
<!-- ios/Runner/Info.plist -->
<key>FIREBASE_ANALYTICS_COLLECTION_ENABLED</key>
<false/>
<key>FirebaseCrashlyticsCollectionEnabled</key>
<false/>
<key>firebase_performance_collection_enabled</key>
<false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_ANALYTICS_STORAGE</key>
<false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_STORAGE</key>
<false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_USER_DATA</key>
<false/>
<key>GOOGLE_ANALYTICS_DEFAULT_ALLOW_AD_PERSONALIZATION_SIGNALS</key>
<false/>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside <application> -->
<meta-data android:name="firebase_analytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_crashlytics_collection_enabled" android:value="false" />
<meta-data android:name="firebase_performance_collection_enabled" android:value="false" />
<meta-data android:name="google_analytics_default_allow_analytics_storage" android:value="false" />
<meta-data android:name="google_analytics_default_allow_ad_storage" android:value="false" />
<meta-data android:name="google_analytics_default_allow_ad_user_data" android:value="false" />
<meta-data android:name="google_analytics_default_allow_ad_personalization_signals" android:value="false" />
```

The `*_deactivated` keys turn a product off permanently and win over any runtime call, so don't use them for consent. On iOS, `GOOGLE_ANALYTICS_IDFV_COLLECTION_ENABLED` set to false stops IDFV collection too.

## Consent & GDPR

GDPR regions need opt-in before any tracking, and the choice has to be as easy to reverse as to give. Store it, apply it on every start, and offer it again in Settings.

```dart
// lib/core/analytics/consent_notifier.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:flutter/foundation.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'consent_notifier.g.dart';

/// A box of its own: other code may open 'settings' with another value type.
const consentBox = 'consent';
const _consentKey = 'analytics_consent';

bool readStoredConsent() =>
    Hive.box<Object>(consentBox).get(_consentKey, defaultValue: false)! as bool;

/// Never on in debug builds, whatever the user chose.
Future<void> applyCollection({required bool consented}) async {
  final enabled = consented && !kDebugMode;
  await FirebaseAnalytics.instance.setConsent(
    analyticsStorageConsentGranted: enabled,
  );
  await FirebaseAnalytics.instance.setAnalyticsCollectionEnabled(enabled);
  await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(enabled);
  await FirebasePerformance.instance.setPerformanceCollectionEnabled(enabled);
}

@Riverpod(keepAlive: true)
class ConsentNotifier extends _$ConsentNotifier {
  @override
  bool build() => readStoredConsent();

  Future<void> grant() => _set(consented: true);

  Future<void> revoke() async {
    await _set(consented: false);
    await FirebaseAnalytics.instance.setUserId();
    await FirebaseCrashlytics.instance.deleteUnsentReports();
  }

  Future<void> _set({required bool consented}) async {
    await Hive.box<Object>(consentBox).put(_consentKey, consented);
    await applyCollection(consented: consented);
    state = consented;
  }
}
```

Crashlytics and Performance apply a change to `false` on the next launch, so say that plainly in your privacy text.

The consent screen shows before any analytics fires, on first launch. It offers clear Accept and Decline options with no pre-ticked box and links to the privacy policy. Decline calls `revoke()`, Accept calls `grant()`, and Settings, Privacy can change the choice at any time. Outside GDPR regions you may default to opted in, but keep the opt-out.

## Firebase Initialization Order

Order matters: Hive opens first because consent reads from it, Firebase next, the error handlers before `runApp`, and consent is applied before any event. Flutter's framework errors go to `FlutterError.onError`, and everything else to `PlatformDispatcher.instance.onError`. That covers what `runZonedGuarded` was once used for, so this skill doesn't use a zone.

```dart
// lib/main.dart
import 'dart:ui';

import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/core/analytics/consent_notifier.dart';
import 'package:yourapp/firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await Hive.openBox<Object>(consentBox);

  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;
  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };

  await applyCollection(consented: readStoredConsent());

  runApp(const ProviderScope(child: MaterialApp(home: Placeholder())));
}
```

## Analytics Event Taxonomy

Widgets never call Analytics directly. Everything goes through a typed service, so a reviewer can read every event in one file, and the consent check sits in one place.

- Use snake_case, with a feature prefix: `auth_`, `ai_`, `purchase_`, `nav_`, `error_`, `engagement_`.
- Limits: 40 characters for an event or parameter name, 100 for a parameter value, 25 parameters per event, 25 user properties, 24 and 36 characters for a user property's name and value.
- No personal data in any value: no email, phone, name, address or device ID.
- Log codes and categories, never free text. A search term or an exception message can hold a person's name or a card number.

```dart
// lib/core/analytics/analytics_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';

enum PurchaseFailureCode { cancelled, declined, network, unknown }

class AnalyticsService {
  AnalyticsService(this._analytics, this._consented);

  final FirebaseAnalytics _analytics;
  final bool Function() _consented;

  Future<void> _log(Future<void> Function() send) async {
    if (_consented()) await send();
  }

  Future<void> logSignIn({required String method}) =>
      _log(() => _analytics.logLogin(loginMethod: method));

  Future<void> logPurchaseFailed({
    required String productId,
    required PurchaseFailureCode code,
  }) =>
      _log(() => _analytics.logEvent(
            name: 'purchase_failed',
            parameters: {'product_id': productId, 'reason': code.name},
          ));

  /// Never the query text: people type names and numbers into search boxes.
  Future<void> logSearchPerformed({
    required String category,
    required int resultCount,
  }) =>
      _log(() => _analytics.logEvent(
            name: 'search_performed',
            parameters: {'category': category, 'result_count': resultCount},
          ));
}
```

`references/event-taxonomy.md` has the full service, with auth, AI, purchase, error and engagement events.

## Screen Tracking with go_router

`FirebaseAnalyticsObserver` logs a `screen_view` on push, replace and pop, using the route's name. go_router names each page `state.name ?? state.path`, so give routes a `name` or you'll report raw patterns such as `:id`. Use one observer: stacking a custom observer on top of it counts every screen twice.

Shell routes need one more step. go_router adds the router's observers to each `StatefulShellBranch` navigator, so the single observer on the `GoRouter` covers pushes inside a branch. But `goBranch` pushes nothing, so log tab switches yourself in `onDestinationSelected`. `references/screen-tracking.md` has the complete router.

## User Properties and a Pseudonymous ID

Google's rule is that a user ID must not hold anything a third party could use to identify the person. An opaque ID from your server is best. If you hash, use SHA-256 with a secret salt from your server: Dart's `hashCode` is not a hash function for this, because it isn't stable across runs or platforms, and an unsalted hash of an email can be reversed with a list of emails.

```dart
// lib/core/analytics/user_properties.dart
import 'dart:convert';

import 'package:crypto/crypto.dart';
import 'package:firebase_analytics/firebase_analytics.dart';

String pseudonymousId(String userId, String serverSalt) =>
    sha256.convert(utf8.encode('$serverSalt:$userId')).toString();

class UserProperties {
  UserProperties(this._analytics);

  final FirebaseAnalytics _analytics;

  Future<void> set({
    required String analyticsId,
    required String subscriptionTier,
    required String locale,
  }) async {
    await _analytics.setUserId(id: analyticsId);
    await _analytics.setUserProperty(
      name: 'subscription_tier',
      value: subscriptionTier,
    );
    await _analytics.setUserProperty(name: 'app_locale', value: locale);
  }

  Future<void> clear() async {
    await _analytics.setUserId();
    await _analytics.setUserProperty(name: 'subscription_tier', value: null);
    await _analytics.setUserProperty(name: 'app_locale', value: null);
  }
}
```

Tier, locale and theme preference are safe. Email, full name, phone and device identifiers never are.

## Crashlytics and Performance

| Task | Where |
|---|---|
| Fatal versus non-fatal, custom keys, breadcrumbs, limits | `references/crashlytics-reporting.md` |
| Dio HTTP traces, custom traces, app startup | `references/performance-monitoring.md` |

## Verify

```dart
// test/core/analytics/user_properties_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/core/analytics/user_properties.dart';

void main() {
  test('the pseudonymous ID is stable', () {
    expect(pseudonymousId('u1', 'salt'), pseudonymousId('u1', 'salt'));
  });

  test('the salt changes the ID', () {
    expect(pseudonymousId('u1', 'a'), isNot(pseudonymousId('u1', 'b')));
  });

  test('the ID hides the input', () {
    expect(pseudonymousId('a@example.com', 's'), isNot(contains('example')));
  });
}
```

Then prove the default is off on a release-like build:

```bash
adb shell setprop debug.firebase.analytics.app com.example.app   # Android DebugView
flutter run --profile     # collection stays off in debug builds by design
```

Decline consent, use the app, and confirm DebugView stays empty. Accept and confirm events arrive. On iOS, add `-FIRDebugEnabled` to the scheme's launch arguments.

```bash
grep -rn "logEvent\|logScreenView" lib/ | grep -v core/analytics   # expect only tab tracking in the router
grep -rn "hashCode" lib/core/analytics                             # expect nothing
```

## Gotchas

1. **Events appear before consent.** Symptom: DebugView shows `first_open` on a fresh install. Cause: native config still has collection on. Fix: set the keys above.
2. **`The setter 'responseCode' isn't defined`.** Symptom: a compile error on `HttpMetric`. Cause: the property is `httpResponseCode`. Fix: rename it.
3. **Every screen is counted twice.** Symptom: `screen_view` is doubled. Cause: two observers on one navigator, which includes one on a branch beside the router's, since go_router merges them. Fix: keep one `FirebaseAnalyticsObserver`, on the `GoRouter`.
4. **Tabs never appear in reports.** Symptom: no `screen_view` on a tab switch. Cause: `goBranch` pushes nothing. Fix: log the switch in `onDestinationSelected`.
5. **`No Firebase App '[DEFAULT]' has been created`.** Symptom: a crash at start. Cause: a Firebase call before `initializeApp`, such as a trace at the top of `main`. Fix: initialize first.
6. **`Box not found`.** Symptom: a crash reading consent. Cause: the box wasn't opened. Fix: open it in `main` before anything reads it.
7. **The user ID changes on each launch.** Symptom: one user counts as many. Cause: `hashCode`. Fix: use a server ID or a salted SHA-256.

## Anti-Patterns

- Consent as a Dart check only. The native SDK runs first.
- Free text in a parameter. Log a code or a category.
- Recording expected failures as non-fatals. It hides real bugs.
- Sending a raw endpoint with IDs in it. Log the route template, such as `/orders/:id`.
- A debug build reporting to production. `applyCollection` keeps it off.

## Pre-Merge Checklist

- [ ] Native keys turn all three products off on both platforms
- [ ] Consent is applied in `main`, before any event
- [ ] Declining shows an empty DebugView
- [ ] Revoking works from Settings
- [ ] No widget calls Analytics directly
- [ ] No parameter holds free text or personal data
- [ ] The user ID is opaque or a salted SHA-256
- [ ] `FlutterError.onError` and `PlatformDispatcher.instance.onError` are both set
- [ ] Debug symbols upload in the release pipeline
- [ ] The privacy policy and Data Safety form match what's collected

## See Also

- `flutter-security` for release symbols and hardening
- `flutter-network` for the Dio client these traces attach to
- `flutter-setup-declarative-routing` for route names
- `flutter-auth` for the auth events
