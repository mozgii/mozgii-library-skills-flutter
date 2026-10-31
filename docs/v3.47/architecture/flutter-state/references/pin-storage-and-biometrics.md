---
title: PIN entry, PIN storage and biometric fallback
author: MOZGII, Inc.
lang: en
tags: [flutter, pin, pinput, security, biometrics, local_auth]
description: PIN entry with pinput 7, storage with salted Argon2id and an attempt limit that ends the PIN after five failures and keeps it locked until sign-out succeeds, the attack tests that prove it, and the biometric to PIN to sign-in fallback chain.
---
# PIN Entry, PIN Storage and Biometric Fallback

An app PIN is a short secret on a device that an attacker may hold. This file treats it that way: a slow salted hash, an attempt limit that survives a restart, and a recovery path that doesn't need the PIN.

## PIN Storage With Salted Argon2id

`PinDataSource` stores `argon2id$memory$iterations$salt$hash` in `flutter_secure_storage`. Each PIN gets 16 random bytes of salt from `Random.secure()`, so two users with the same PIN have different hashes, and a table built in advance is useless. The cost is stored with the hash, so a later release can raise it for new PINs without breaking old ones.

`Argon2id` from `cryptography` runs in pure Dart, so it goes in `Isolate.run`: on the UI isolate it would freeze the screen. The starting cost is OWASP's minimum for Argon2id: 19 MiB of memory (`memory: 19456`, counted in KiB), 2 iterations, 1 degree of parallelism.

## Tune the PIN Check to Half a Second

The target for an unlock is about half a second for one check, timed in a release build on the slowest phone you support. OWASP's limit for any password hash is under one second, so treat a second as the ceiling. In a release-mode build on an Apple M5 Max the minimum cost took 193 ms, and a phone is slower than that.

If a check finishes well under the target, raise `memory` until it reaches it. Never go below 19 MiB at 2 iterations to save time: OWASP's other rows, such as 12 MiB at 3 iterations, trade memory for CPU, so they spend more time rather than less.

A slower hash has a limit of its own, because the KDF isn't the main defence. It only slows an attacker who has already pulled the hash out of the platform keystore, and at one million possible PINs no cost would hold for long. Secure storage and the attempt limit do most of the work.

## Enforcing the PIN Attempt Limit

The limit is what stops a guesser, so it follows three rules:

1. **The count is persisted**, in secure storage beside the hash. Killing the app resets nothing.
2. **The count goes up before the check.** If it went up after, killing the process during the half-second KDF would give a free guess.
3. **Five failures end the PIN.** Every later call answers `PinRemoved` before the KDF runs, even for the correct PIN, and the app has to send the person through a full sign-in. The hash stays in storage until that sign-out succeeds, so a kill and reopen with no network still finds a lock. Only a correct PIN resets the count, and nothing here reads the clock, so changing the device's time gains an attacker nothing.

Calls run one at a time through `_serial`, so two taps can't race on the counter.

```dart
// lib/features/auth/data/datasources/pin_datasource.dart
import 'dart:convert';
import 'dart:isolate';
import 'dart:math';

import 'package:cryptography/cryptography.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

sealed class PinResult {
  const PinResult();
}

class PinAccepted extends PinResult {
  const PinAccepted();
}

class PinRejected extends PinResult {
  const PinRejected(this.attemptsLeft);
  final int attemptsLeft;
}

/// Five failures: this PIN opens nothing now, and the person must sign in again.
/// The hash stays until sign-out succeeds, so a failed sign-out stays locked.
class PinRemoved extends PinResult {
  const PinRemoved();
}

Future<List<int>> _derive(
  String pin,
  List<int> salt,
  int memory,
  int iterations,
) => Isolate.run(() async {
  final argon2id = Argon2id(
    parallelism: 1,
    memory: memory,
    iterations: iterations,
    hashLength: 32,
  );
  final key = await argon2id.deriveKeyFromPassword(password: pin, nonce: salt);
  return key.extractBytes();
});

bool _sameBytes(List<int> a, List<int> b) {
  if (a.length != b.length) return false;
  var difference = 0;
  for (var i = 0; i < a.length; i++) {
    difference |= a[i] ^ b[i];
  }
  return difference == 0;
}

class PinDataSource {
  PinDataSource(this._storage, {this.memory = 19456, this.iterations = 2});

  final FlutterSecureStorage _storage;
  final int memory;
  final int iterations;

  static const maxFailures = 5;
  static const _pinKey = 'app_pin_hash';
  static const _failuresKey = 'app_pin_failures';

  Future<void> _tail = Future<void>.value();

  Future<T> _serial<T>(Future<T> Function() action) {
    final result = _tail.then((_) => action());
    _tail = result.then<void>((_) {}, onError: (Object _) {});
    return result;
  }

  Future<void> setPin(String pin) => _serial(() async {
    final random = Random.secure();
    final salt = List<int>.generate(16, (_) => random.nextInt(256));
    final hash = await _derive(pin, salt, memory, iterations);
    final stored =
        'argon2id\$$memory\$$iterations'
        '\$${base64Encode(salt)}\$${base64Encode(hash)}';
    await _storage.write(key: _pinKey, value: stored);
    await _storage.delete(key: _failuresKey);
  });

  Future<bool> hasPin() async => await _storage.read(key: _pinKey) != null;

  Future<PinResult> verifyPin(String pin) => _serial(() async {
    final stored = await _storage.read(key: _pinKey);
    if (stored == null) return const PinRemoved();

    final used = int.parse(await _storage.read(key: _failuresKey) ?? '0');
    if (used >= maxFailures) return const PinRemoved();

    final failures = used + 1;
    await _storage.write(key: _failuresKey, value: '$failures');

    final parts = stored.split(r'$');
    final hash = await _derive(
      pin,
      base64Decode(parts[3]),
      int.parse(parts[1]),
      int.parse(parts[2]),
    );
    if (_sameBytes(hash, base64Decode(parts[4]))) {
      await _storage.delete(key: _failuresKey);
      return const PinAccepted();
    }

    return failures >= maxFailures
        ? const PinRemoved()
        : PinRejected(maxFailures - failures);
  });

  /// Called through `pinConfiguredProvider.notifier`. At the attempt limit,
  /// only after sign-out has succeeded.
  Future<void> clearPin() async {
    await _storage.delete(key: _pinKey);
    await _storage.delete(key: _failuresKey);
  }
}
```

## Attack Tests for the PIN Store

These tests attack the store instead of exercising it. They use a low cost for speed, and they build a new `PinDataSource` to stand in for a restarted app.

```dart
// test/features/auth/pin_datasource_test.dart
import 'package:app/features/auth/data/datasources/pin_datasource.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  PinDataSource store() =>
      PinDataSource(const FlutterSecureStorage(), memory: 64, iterations: 1);

  setUp(() => FlutterSecureStorage.setMockInitialValues({}));

  test('the same PIN gets a different salt and hash each time', () async {
    const storage = FlutterSecureStorage();
    await store().setPin('123456');
    final first = await storage.read(key: 'app_pin_hash');
    await store().setPin('123456');
    final second = await storage.read(key: 'app_pin_hash');
    expect(first, isNot(second));
    expect(first, isNot(contains('123456')));
    expect(first, startsWith(r'argon2id$64$1$'));
  });

  test('the correct PIN passes and resets the counter', () async {
    await store().setPin('123456');
    expect(await store().verifyPin('000000'), isA<PinRejected>());
    expect(await store().verifyPin('123456'), isA<PinAccepted>());
    expect(
      await const FlutterSecureStorage().read(key: 'app_pin_failures'),
      isNull,
    );
  });

  test('a restart keeps the count, and five failures end the PIN', () async {
    await store().setPin('123456');
    for (var left = PinDataSource.maxFailures - 1; left > 0; left--) {
      expect(
        await store().verifyPin('000000'),
        isA<PinRejected>().having((r) => r.attemptsLeft, 'left', left),
      );
    }
    expect(await store().verifyPin('000000'), isA<PinRemoved>());
    expect(await store().hasPin(), isTrue); // kept until sign-out succeeds
    expect(await store().verifyPin('123456'), isA<PinRemoved>());
  });

  test('past the limit no guess reaches the KDF', () async {
    await store().setPin('123456');
    for (var i = 0; i < PinDataSource.maxFailures; i++) {
      await store().verifyPin('000000');
    }
    // Parsing this value would throw, so an answer proves no derivation ran.
    await const FlutterSecureStorage().write(
      key: 'app_pin_hash',
      value: 'not-a-hash',
    );
    expect(await store().verifyPin('123456'), isA<PinRemoved>());
  });

  test('parallel attempts cannot slip past the limit', () async {
    final one = store();
    await one.setPin('123456');
    final results = await Future.wait(
      List.generate(
        PinDataSource.maxFailures + 2,
        (_) => one.verifyPin('000000'),
      ),
    );
    expect(results.whereType<PinRemoved>(), hasLength(3));
  });
}
```

## PIN Entry Widget With pinput 7

`Pinput` from `pinput` 7 draws the boxes. A security PIN isn't a text-message code, so `autofillHints` is empty and suggestions are off. Pinput shows `errorText` itself, so the widget doesn't add a second `Text`; `forceErrorState` turns the error colours on.

Pinput removed `androidSmsAutofillMethod` and `listenForMultipleSmsOnAndroid`. When a screen really does take a one-time code from a text message, pass your own `SmsRetriever` to the `smsRetriever` parameter; it has `listenForMultipleSms`, `getSmsCode()` and `dispose()`.

```dart
// lib/features/auth/presentation/widgets/pin_entry.dart
import 'package:material_ui/material_ui.dart';
import 'package:pinput/pinput.dart';

class PinEntryWidget extends StatelessWidget {
  const PinEntryWidget({
    required this.onCompleted,
    this.controller,
    this.errorText,
    this.enabled = true,
    super.key,
  });

  final void Function(String pin) onCompleted;
  final TextEditingController? controller;
  final String? errorText;
  final bool enabled;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final defaultPinTheme = PinTheme(
      width: 56,
      height: 60,
      textStyle: theme.textTheme.headlineSmall?.copyWith(
        color: theme.colorScheme.onSurface,
      ),
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(12),
        border: Border.all(color: theme.colorScheme.outline),
      ),
    );

    return Pinput(
      controller: controller,
      length: 6,
      enabled: enabled,
      autofocus: true,
      defaultPinTheme: defaultPinTheme,
      focusedPinTheme: defaultPinTheme.copyWith(
        decoration: defaultPinTheme.decoration!.copyWith(
          border: Border.all(color: theme.colorScheme.primary, width: 2),
        ),
      ),
      errorPinTheme: defaultPinTheme.copyWith(
        decoration: defaultPinTheme.decoration!.copyWith(
          border: Border.all(color: theme.colorScheme.error),
        ),
      ),
      errorText: errorText,
      forceErrorState: errorText != null,
      obscureText: true,
      obscuringCharacter: '●',
      hapticFeedbackType: HapticFeedbackType.lightImpact,
      autofillHints: const [],
      enableSuggestions: false,
      closeKeyboardWhenCompleted: false,
      onCompleted: onCompleted,
    );
  }
}
```

## Lock Screen: Reading Every PIN Result

The lock screen turns each `PinResult` into something the person can act on. It shows a spinner while the KDF runs, and clears the field after a rejection so the next attempt starts empty. `PinRemoved` ends the session first and only then deletes the hash, through `pinConfiguredProvider.notifier.clearPin()`, which lifts the lock, so the router lands on sign-in with nothing to show. If the sign-out fails, nothing is deleted: the screen stays locked and says so, a kill and reopen finds the hash and locks again, and the next entry tries the sign-out again.

```dart
// lib/features/auth/presentation/pages/app_lock_screen.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:material_ui/material_ui.dart';

import '../../../../core/app.dart';
import '../../data/datasources/pin_datasource.dart';
import '../widgets/pin_entry.dart';

final pinDataSourceProvider = Provider(
  (ref) => PinDataSource(const FlutterSecureStorage()),
);

/// Bind this to your auth repository: clear tokens and end the session.
final signOutProvider = Provider<Future<void> Function()>(
  (ref) => throw UnimplementedError('Provide the sign-out'),
);

class AppLockScreen extends ConsumerStatefulWidget {
  const AppLockScreen({super.key});

  @override
  ConsumerState<AppLockScreen> createState() => _AppLockScreenState();
}

class _AppLockScreenState extends ConsumerState<AppLockScreen> {
  final _pin = TextEditingController();
  String? _error;
  bool _checking = false;

  @override
  void dispose() {
    _pin.dispose();
    super.dispose();
  }

  Future<void> _check(String pin) async {
    setState(() => _checking = true);
    final result = await ref.read(pinDataSourceProvider).verifyPin(pin);
    if (!mounted) return;
    if (result is! PinAccepted) _pin.clear();
    setState(() {
      _checking = false;
      _error = switch (result) {
        PinAccepted() => null,
        PinRejected(:final attemptsLeft) =>
          'Wrong PIN. $attemptsLeft attempts left.',
        PinRemoved() => 'PIN removed. Sign in again.',
      };
    });
    switch (result) {
      case PinAccepted():
        ref.read(appLockProvider.notifier).unlock();
      case PinRejected():
        break;
      case PinRemoved():
        await _signOutAndUnlock();
    }
  }

  Future<void> _signOutAndUnlock() async {
    try {
      await ref.read(signOutProvider)();
      if (mounted) await ref.read(pinConfiguredProvider.notifier).clearPin();
    } on Exception {
      if (mounted) {
        setState(() {
          _error = 'Could not sign out. Check your connection, then enter any '
              'PIN to try again.';
        });
      }
    }
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    body: Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text('Enter your PIN', style: Theme.of(context).textTheme.titleLarge),
          const SizedBox(height: 24),
          PinEntryWidget(
            controller: _pin,
            onCompleted: _check,
            errorText: _error,
            enabled: !_checking,
          ),
          if (_checking) const LinearProgressIndicator(),
        ],
      ),
    ),
  );
}
```

## The Lock Follows the Stored PIN

A running app sets and removes a PIN through `pinConfiguredProvider.notifier`, never on `PinDataSource` directly, so the lock always knows. These tests prove both directions: removing the PIN lifts a lock, and a new PIN waits for the next resume instead of locking the person out mid-setup. The last two drive the real lock screen at the attempt limit: a sign-out that fails in airplane mode leaves the app locked across a restart, and one that succeeds deletes the hash and lifts the lock.

```dart
// test/core/app_lock_test.dart
import 'package:app/core/app.dart';
import 'package:app/features/auth/data/datasources/pin_datasource.dart';
import 'package:app/features/auth/presentation/pages/app_lock_screen.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

class _MemoryPins extends PinDataSource {
  _MemoryPins() : super(const FlutterSecureStorage());

  @override
  Future<void> setPin(String pin) async {}

  @override
  Future<void> clearPin() async {}
}

void main() {
  ProviderContainer app({required bool hasPin}) {
    final container = ProviderContainer(
      overrides: [
        pinDataSourceProvider.overrideWithValue(_MemoryPins()),
        pinConfiguredProvider.overrideWithBuild((_, _) => hasPin),
      ],
    );
    addTearDown(container.dispose);
    return container;
  }

  test('a stored PIN starts locked, and removing it lifts the lock', () async {
    final container = app(hasPin: true);
    expect(container.read(appLockProvider), isTrue);

    await container.read(pinConfiguredProvider.notifier).clearPin();

    expect(container.read(appLockProvider), isFalse);
    expect(container.read(pinConfiguredProvider), isFalse);
  });

  test('a new PIN counts at once and locks only on resume', () async {
    final container = app(hasPin: false);
    expect(container.read(appLockProvider), isFalse);

    await container.read(pinConfiguredProvider.notifier).setPin('123456');

    expect(container.read(pinConfiguredProvider), isTrue);
    expect(container.read(appLockProvider), isFalse);
  });

  group('at the attempt limit', () {
    PinDataSource store() =>
        PinDataSource(const FlutterSecureStorage(), memory: 64, iterations: 1);

    /// Spends all but one attempt, then enters a wrong PIN on the lock screen.
    Future<ProviderContainer> failLastAttempt(
      WidgetTester tester,
      Future<void> Function() signOut,
    ) async {
      FlutterSecureStorage.setMockInitialValues({});
      await tester.runAsync(() async {
        await store().setPin('123456');
        for (var i = 1; i < PinDataSource.maxFailures; i++) {
          await store().verifyPin('000000');
        }
      });
      final container = ProviderContainer(
        overrides: [
          pinDataSourceProvider.overrideWithValue(store()),
          pinConfiguredProvider.overrideWithBuild((_, _) => true),
          signOutProvider.overrideWithValue(signOut),
        ],
      );
      addTearDown(container.dispose);
      await tester.pumpWidget(
        UncontrolledProviderScope(
          container: container,
          child: const MaterialApp(home: AppLockScreen()),
        ),
      );
      await tester.enterText(find.byType(EditableText), '000000');
      await tester.runAsync(
        () => Future<void>.delayed(const Duration(milliseconds: 200)),
      );
      await tester.pump();
      return container;
    }

    testWidgets('a failed sign-out, then a restart, is still locked', (
      tester,
    ) async {
      final container = await failLastAttempt(
        tester,
        () async => throw Exception('airplane mode'),
      );
      expect(find.textContaining('Could not sign out'), findsOneWidget);
      expect(container.read(appLockProvider), isTrue);

      // A restart sees only what is in storage.
      final restarted = await tester.runAsync(
        () async => (
          hasPin: await store().hasPin(),
          result: await store().verifyPin('123456'),
        ),
      );
      expect(restarted!.hasPin, isTrue);
      expect(restarted.result, isA<PinRemoved>());
    });

    testWidgets('a sign-out that works deletes the hash and lifts the lock', (
      tester,
    ) async {
      final container = await failLastAttempt(tester, () async {});
      expect(container.read(appLockProvider), isFalse);
      expect(await tester.runAsync(store().hasPin), isFalse);
    });
  });
}
```

## Biometric and PIN Fallback Hierarchy

The chain runs biometric, then PIN, then an account reset link. Biometrics never unlock anything directly in the PIN store: a success just skips the PIN. Cancelling or failing the prompt moves on to the PIN, so a person with a wet finger isn't stuck. Since `local_auth` 3, failures throw `LocalAuthException` with a `LocalAuthExceptionCode`, not a `PlatformException`.

```dart
// lib/features/auth/domain/usecases/local_auth_usecase.dart
import 'package:local_auth/local_auth.dart';

import '../../data/datasources/pin_datasource.dart';

enum LocalAuthOutcome { authenticated, requiresPin, notConfigured }

class BiometricDataSource {
  BiometricDataSource(this._auth);
  final LocalAuthentication _auth;

  Future<bool> isAvailable() async =>
      await _auth.isDeviceSupported() && await _auth.canCheckBiometrics;

  Future<bool> authenticate({required String reason}) async {
    try {
      return await _auth.authenticate(
        localizedReason: reason,
        biometricOnly: true,
        persistAcrossBackgrounding: true,
      );
    } on LocalAuthException {
      return false;
    }
  }
}

class LocalAuthUseCase {
  const LocalAuthUseCase(this._biometric, this._pin);

  final BiometricDataSource _biometric;
  final PinDataSource _pin;

  Future<LocalAuthOutcome> authenticate() async {
    // Step 1: try biometric
    if (await _biometric.isAvailable()) {
      final success = await _biometric.authenticate(
        reason: 'Verify your identity to continue',
      );
      if (success) return LocalAuthOutcome.authenticated;
    }

    // Step 2: fall back to PIN if set, so the UI shows PIN entry
    if (await _pin.hasPin()) return LocalAuthOutcome.requiresPin;

    // Step 3: no local auth configured, so offer the account reset link
    return LocalAuthOutcome.notConfigured;
  }
}
```
