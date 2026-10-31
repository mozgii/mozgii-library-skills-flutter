---
name: flutter-state
description: Builds the app-wide state layer of a Flutter app - Riverpod keepAlive providers, connectivity with a real internet probe, dark and light theme with Hive persistence and zero flash at startup, PIN entry with pinput 7, PIN storage with salted Argon2id and an enforced attempt limit, biometric fallback, PIN lock on resume and cold-start hydration. Use when state must survive navigation, when adding a dark mode toggle, offline handling, an app lock or a PIN, or when restoring state from local storage at startup.
metadata:
  tags:
  - flutter
  - state
  - connectivity
  - dark-mode
  - theme
  - pin
  - offline
  - hydration
---
# Flutter App State

This skill covers the state that outlives any one screen: the theme, whether the device is really online, whether the app is locked, and what's restored at startup. Getting these wrong shows up as a white flash at launch, a spinner on a captive portal, or a PIN that a killed app forgets to ask for. Use it for global state that survives page navigation, a persisted dark mode toggle, a PIN re-entry guard, online and offline transitions, or hydration at cold start. Skip it when state is scoped to one feature, or when you're only building the visual theme, which `flutter-design-ui` covers.

## Step 0: Read Your Versions

```bash
flutter --version
flutter pub deps --style=compact | grep -E "^- (flutter_riverpod|hive_ce|pinput|connectivity_plus|local_auth|material_ui) "
grep -rnE "valueOrNull|androidSmsAutofillMethod|sha256\.convert" lib     # old APIs and an unsafe PIN hash
```

The pins below were read from pub.dev on 2026-10-30 and the examples were compiled against them with Flutter 3.47.9 and Dart 3.13.7. `pinput` 7 needs `material_ui`, so migrate first with `dart fix --apply --code=migrate_design_widgets`. `build_runner: ^2.16.1` sits below the latest 2.17.1 on purpose: 2.17.1 needs `build` 5, which `envied_generator`, `hive_ce_generator`, `injectable_generator`, `isar_community_generator` and `riverpod_generator` don't accept yet, and 2.16.2 needs `analyzer` 14.3, which `injectable_generator` 3.1.3 can't take. `connectivity_plus: ^7.3.1` sits below the latest 7.3.2 because 7.3.2 needs `dbus` 0.8 while `geolocator` 14.1.2 needs 0.7. Everything here was compiled, not run on a device, so try the lock, PIN and biometric flow on real hardware.

```yaml
dependencies:
  connectivity_plus: ^7.3.1    # below 7.3.2: its dbus 0.8 conflicts with geolocator
  cryptography: ^2.9.0
  dynamic_color: ^2.1.0
  flutter_hooks: ^0.21.3+1
  flutter_riverpod: ^3.4.3
  flutter_secure_storage: ^11.2.0
  freezed_annotation: ^3.1.0
  go_router: ^18.1.2
  hive_ce: ^2.22.0
  hive_ce_flutter: ^2.5.0
  hooks_riverpod: ^3.4.3
  http: ^1.6.0
  local_auth: ^3.0.2
  material_ui: ^1.9.0
  pinput: ^7.0.0
  riverpod_annotation: ^4.0.7

dev_dependencies:
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
  freezed: ^4.0.3
  hive_ce_generator: ^1.12.0
  riverpod_generator: ^4.0.9
```

## Choosing Where State Lives

| State | Use | Why the others lose |
|---|---|---|
| Survives navigation, app-wide | `@Riverpod(keepAlive: true)` | `InheritedWidget` needs a `BuildContext`; a plain singleton can't notify widgets |
| One screen or feature | `@riverpod` (auto-dispose) | `keepAlive` holds memory for a screen nobody's on |
| Small preference read before the first frame | Hive box, opened in `main()` | SharedPreferences reads are async, which is what causes a theme flash |
| PIN hash, tokens, anything secret | `flutter_secure_storage` | Hive and SharedPreferences are plain files in a backup |
| Is the device online? | Interface check + HTTP probe | The interface says Wi-Fi on a captive portal with no internet |

| App lock | Use when | Limit |
|---|---|---|
| Biometric, then PIN, then sign-in | Most apps: fast, and it has a recovery path | Needs a PIN screen as well |
| PIN only | Devices without biometrics, or a shared-device rule | A short secret, so the attempt limit carries the security |
| Device credentials through `local_auth` | No custom UI wanted | No app-specific PIN and no way to keep it separate from the device |

## Theme Mode With Zero Flash on Cold Start

A dark mode preference must be readable before the first frame. Open the Hive box in `main()`, before `runApp()`, and `ThemeModeNotifier.build()` can read it synchronously. That gives zero flash: the first frame already has the right theme.

```dart
// lib/core/state/theme_notifier.dart
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:material_ui/material_ui.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'theme_notifier.g.dart';

@Riverpod(keepAlive: true)
class ThemeModeNotifier extends _$ThemeModeNotifier {
  @override
  ThemeMode build() => switch (Hive.box<String>('settings').get('theme_mode')) {
    'light' => ThemeMode.light,
    'dark' => ThemeMode.dark,
    _ => ThemeMode.system,
  };

  Future<void> setTheme(ThemeMode mode) async {
    state = mode;
    final key = switch (mode) {
      ThemeMode.light => 'light',
      ThemeMode.dark => 'dark',
      ThemeMode.system => 'system',
    };
    await Hive.box<String>('settings').put('theme_mode', key);
  }

  Future<void> toggleTheme() =>
      setTheme(state == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark);
}
```

A toggle button with an accessible label is in `references/theme-toggle.md`.

## Network Connectivity With a Real Internet Probe

`connectivity_plus` reports interface type only, and a device can show Wi-Fi with no internet on a captive portal. So use a two-step check: the interface, then a lightweight HTTP probe of your own endpoint. Over HTTPS a portal can't fake the answer, and the probe accepts only a 2xx, because a portal's redirect isn't internet. The stream and `checkConnectivity()` both give a `List<ConnectivityResult>`, since a device can be on Wi-Fi and a VPN at once. The list is never empty, and `none` appears only alone. Replies can arrive out of order on a flapping network, so a counter drops any probe that a newer event has overtaken.

```dart
// lib/core/state/connectivity_notifier.dart
import 'dart:async';

import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:http/http.dart' as http;
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'connectivity_notifier.freezed.dart';
part 'connectivity_notifier.g.dart';

@freezed
sealed class ConnectivityState with _$ConnectivityState {
  const factory ConnectivityState.unknown() = ConnectivityStateUnknown;
  const factory ConnectivityState.online({required String type}) =
      ConnectivityStateOnline;
  const factory ConnectivityState.offline() = ConnectivityStateOffline;
}

@Riverpod(keepAlive: true)
class ConnectivityNotifier extends _$ConnectivityNotifier {
  int _latest = 0;

  @override
  ConnectivityState build() {
    final connectivity = Connectivity();
    final sub = connectivity.onConnectivityChanged.listen(_apply);
    ref.onDispose(sub.cancel);
    connectivity.checkConnectivity().then(_apply);
    return const ConnectivityState.unknown();
  }

  Future<void> _apply(List<ConnectivityResult> results) async {
    final current = ++_latest;
    if (results.contains(ConnectivityResult.none)) {
      state = const ConnectivityState.offline();
      return;
    }
    final hasInternet = await _probe();
    if (current != _latest) return;
    state = hasInternet
        ? ConnectivityState.online(type: results.first.name)
        : const ConnectivityState.offline();
  }

  /// Lightweight internet reachability probe.
  Future<bool> _probe() async {
    try {
      final response = await http
          .head(Uri.parse('https://api.example.com/health'))
          .timeout(const Duration(seconds: 3));
      return response.statusCode >= 200 && response.statusCode < 300;
    } on Exception {
      return false;
    }
  }
}
```

Refetching when the network returns, and the offline banner, are in `references/connectivity-ui.md`.

## Worked Example: Lock, Theme and Router in One App Shell

`MainApp` joins the state above into one shell. It watches the theme and the lock, keeps the router alive under the lock, and starts locked whenever a PIN exists.

### PIN Lock on App Resume

`AppLifecycleListener` gives named callbacks, so nothing here needs `didChangeAppLifecycleState`. The time is recorded in `onHide` rather than `onPause`, because `paused` is entered on iOS and Android alone, while every platform reaches `hidden`. The device clock can be set back to shorten the time away, so a monotonic `Stopwatch` runs beside it and the app locks when either passes the limit. The wall clock stays because a `Stopwatch` may stop while the device sleeps. A negative elapsed time means the clock was moved back, which locks too.

The cold-start case matters just as much. A lock that starts as `false` is skipped by killing the app and reopening it, so `main()` seeds `pinConfiguredProvider` and the lock starts on when a PIN exists. That provider is a notifier, and its `setPin` and `clearPin` are how a running app changes the PIN, so a PIN removed mid-session lifts the lock and a new one locks on the next resume. A value fixed at startup gets both wrong. The router stays in the tree under `Offstage`, so unlocking returns to the same screen. `Overlay.wrap` is there because `MaterialApp.builder` sits above the Navigator, and a text field needs an overlay.

```dart
// lib/core/app.dart
import 'package:dynamic_color/dynamic_color.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../features/auth/presentation/pages/app_lock_screen.dart';
import 'state/theme_notifier.dart';

part 'app.g.dart';

@Riverpod(keepAlive: true)
GoRouter router(Ref ref) => GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (_, _) => const Scaffold(body: Center(child: Text('Home'))),
    ),
  ],
);

/// Whether a PIN is stored. main() seeds it, and each change goes through here.
@Riverpod(keepAlive: true)
class PinConfigured extends _$PinConfigured {
  @override
  bool build() => false;

  Future<void> setPin(String pin) async {
    await ref.read(pinDataSourceProvider).setPin(pin);
    state = true;
  }

  Future<void> clearPin() async {
    await ref.read(pinDataSourceProvider).clearPin();
    state = false;
  }
}

@Riverpod(keepAlive: true)
class AppLockNotifier extends _$AppLockNotifier {
  @override
  bool build() {
    // A removed PIN can't be entered, so the lock goes with it.
    ref.listen(pinConfiguredProvider, (_, configured) {
      if (!configured) state = false;
    });
    return ref.read(pinConfiguredProvider); // read: a new PIN locks on resume
  }

  void lock() => state = true;
  void unlock() => state = false;
}

class MainApp extends ConsumerStatefulWidget {
  const MainApp({super.key});

  @override
  ConsumerState<MainApp> createState() => _MainAppState();
}

class _MainAppState extends ConsumerState<MainApp> {
  late final AppLifecycleListener _lifecycleListener;
  DateTime? _hiddenAt;
  final _hiddenFor = Stopwatch();
  static const _lockAfter = Duration(seconds: 60);

  @override
  void initState() {
    super.initState();
    _lifecycleListener = AppLifecycleListener(
      onHide: () {
        _hiddenAt = DateTime.now();
        _hiddenFor
          ..reset()
          ..start();
      },
      onResume: () {
        final hiddenAt = _hiddenAt;
        _hiddenAt = null;
        _hiddenFor.stop();
        if (hiddenAt == null) return;
        final elapsed = DateTime.now().difference(hiddenAt);
        if (elapsed.isNegative ||
            elapsed > _lockAfter ||
            _hiddenFor.elapsed > _lockAfter) {
          if (ref.read(pinConfiguredProvider)) {
            ref.read(appLockProvider.notifier).lock();
          }
        }
      },
    );
  }

  @override
  void dispose() {
    _lifecycleListener.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final isLocked = ref.watch(appLockProvider);
    final themeMode = ref.watch(themeModeProvider);
    return DynamicColorBuilder(
      builder: (lightDynamic, darkDynamic) => MaterialApp.router(
        theme: ThemeData(colorScheme: lightDynamic),
        darkTheme: ThemeData(
          colorScheme: darkDynamic,
          brightness: darkDynamic == null ? Brightness.dark : null,
        ),
        themeMode: themeMode,
        localizationsDelegates: GlobalMaterialLocalizations.delegates,
        routerConfig: ref.watch(routerProvider),
        builder: (context, child) => Stack(
          children: [
            Offstage(
              offstage: isLocked,
              child: child ?? const SizedBox.shrink(),
            ),
            if (isLocked) Overlay.wrap(child: const AppLockScreen()),
          ],
        ),
      ),
    );
  }
}
```

### Startup in main()

`main()` opens the settings box and reads the PIN flag before `runApp()`. Everything slower belongs to the hydration gate in `references/cold-start-hydration.md`.

```dart
// lib/main.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:material_ui/material_ui.dart';

import 'core/app.dart';
import 'core/state/startup_gate.dart';
import 'features/auth/data/datasources/pin_datasource.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await Hive.openBox<String>('settings');
  final hasPin = await PinDataSource(const FlutterSecureStorage()).hasPin();

  runApp(
    ProviderScope(
      overrides: [pinConfiguredProvider.overrideWithBuild((_, _) => hasPin)],
      child: const StartupGate(),
    ),
  );
}
```

## PIN Security Rules

A PIN is never stored in plaintext, and never as a plain or unsalted hash. A six-digit PIN has one million values, so a fast hash such as SHA-256 falls to a table in a moment. Each PIN gets a random salt and Argon2id at OWASP's minimum or above: 19 MiB of memory, 2 iterations, 1 degree of parallelism. Tune it so one unlock takes about half a second in a release build on your slowest supported phone, with one second as the ceiling. The hash lives in `flutter_secure_storage`, derivation runs in an isolate, and the attempt count lives beside the hash so a restart can't reset it. The fifth failure ends the PIN and sends the person through a full sign-in, and the hash stays until that sign-out succeeds, so a failed sign-out stays locked. A KDF alone can't save six digits: the secure storage and the attempt limit do most of the work. The full code, its measurements and its attack tests are in `references/pin-storage-and-biometrics.md`.

## Verify

```bash
dart run build_runner build
flutter analyze
flutter test
grep -rn "sha256.convert\|\.first == ConnectivityResult" lib     # expect nothing
```

Check these on a device, because they depend on the OS:

1. Set dark mode, kill the app and open it: no light frame appears.
2. Set a PIN, kill the app and open it: the lock screen shows first. If it doesn't, the cold-start path is broken.
3. Wait past the timeout in the background: the lock shows, and unlocking returns to the same screen. Repeat, and before resuming set the device clock back to within the timeout of when you left: the lock still shows.
4. Enter a wrong PIN four times, then kill and reopen the app: one more wrong PIN ends it and signs you out. Repeat in airplane mode: the sign-out fails, the lock screen says so, and after a kill and reopen the app is still locked until a sign-out succeeds.
5. Join a Wi-Fi network with no internet: the state goes offline within the probe timeout.

## Gotchas

1. **The app opens without the PIN after being killed.** Symptom: swiping the app away skips the lock. Cause: the lock starts `false`, and only resume sets it. Fix: seed it from whether a PIN exists, as `pinConfiguredProvider` does.
2. **Unlocking lands on the home screen.** Symptom: the user loses their place after every lock. Cause: `builder` returned the lock screen instead of `child`, which dropped the Navigator. Fix: keep `child` in a `Stack` under `Offstage`.
3. **`No Overlay widget found` on the lock screen.** Symptom: the PIN field throws when focused. Cause: `MaterialApp.builder` sits above the Navigator's overlay. Fix: wrap the lock screen in `Overlay.wrap`.
4. **`The named parameter 'androidSmsAutofillMethod' isn't defined`.** Symptom: an old Pinput example won't compile. Cause: `Pinput` 7 has no such parameters; SMS now goes through the `smsRetriever` parameter and your own `SmsRetriever`. Fix: remove them.
5. **`isn't a valid override of 'State.build'`.** Symptom: `build(BuildContext context, WidgetRef ref)` fails on a `ConsumerState` class. Cause: only `ConsumerWidget.build` takes `ref`; `ConsumerState` already has it as a field. Fix: declare `build(BuildContext context)`.
6. **The app stays online on a captive portal.** Symptom: the banner never shows and requests time out. Cause: the check stopped at the interface. Fix: add the probe.
7. **A white frame before dark mode.** Symptom: a flash at launch. Cause: the theme was read asynchronously after `runApp()`. Fix: open the Hive box in `main()`.
8. **A frozen UI during a PIN check.** Symptom: jank on submit. Cause: the KDF ran on the UI isolate. Fix: run it in `Isolate.run`, as the reference does.
9. **The PIN field offers a text-message code.** Symptom: the keyboard suggests a one-time code for a PIN. Cause: Pinput's `autofillHints` defaults to `AutofillHints.oneTimeCode`. Fix: pass `autofillHints: const []`.
10. **The lock asks for a PIN that was just removed.** Symptom: after turning the PIN off, the lock returns on resume and only sign-out gets past it. Cause: whether a PIN exists was read once at startup. Fix: change it through `pinConfiguredProvider.notifier`.

## Anti-Patterns

| Don't | Do instead |
|---|---|
| Use `InheritedWidget` or a singleton for global state | `@Riverpod(keepAlive: true)` |
| Trust `connectivity_plus` alone | Interface check plus HTTP probe |
| Read `results.first` to detect offline | `results.contains(ConnectivityResult.none)` |
| Store a PIN with SHA-256 or without a salt | Salted Argon2id in secure storage |
| Keep the attempt count in memory | Persist it, and write it before checking the PIN |
| Call `deleteAll()` at the attempt limit | Delete only the PIN keys |
| Replace `child` with the lock screen | `Offstage` over a `Stack` |
| Stay on the splash screen when hydration fails | Show an error with a retry |

## Pre-Merge Checklist

- [ ] The settings box opens before `runApp()`
- [ ] Offline detection uses a probe of your own endpoint over HTTPS
- [ ] The PIN uses salted Argon2id, derives in an isolate, and one check takes about half a second on the slowest supported phone
- [ ] The attempt counter persists and is written before the comparison
- [ ] The app locks at cold start when a PIN exists, and removing the PIN lifts the lock
- [ ] Icon-only controls have a tooltip, and the offline banner is a live region
- [ ] Hydration failure has a visible retry, and corrupt cache can't brick startup
- [ ] The device checks in Verify passed on iOS and Android

## See Also

`flutter-core` (Riverpod setup, routing), `flutter-auth` (sign-in, sessions), `flutter-security` (secure storage, screenshots in the app switcher), `flutter-database` (Hive CE and offline storage), `flutter-theme` (colour schemes). Depth in this skill: `references/theme-toggle.md`, `references/connectivity-ui.md`, `references/pin-storage-and-biometrics.md`, `references/cold-start-hydration.md`.
