---
title: "Flutter PIN lock: Argon2id, attempt limit and fallback"
author: MOZGII, Inc.
lang: en
tags: [flutter, auth, pin, argon2, security]
description: An app-level PIN for re-entry - Argon2id with a per-user salt, an attempt limit that survives restarts, the entry screen with pinput, and the biometric to PIN to full sign-in fallback.
---
# PIN Lock for a Flutter App

A PIN reopens an app for someone who is already signed in; it never replaces their account password. Six digits give only a million values, so no hash can make a PIN strong on its own. A salted slow KDF slows an offline attacker with a copy of the store, and the attempt limit is what stops guessing on the device.

## Hash the PIN with Argon2id

Plain SHA-256 is the wrong tool here: it's built to be fast, and with no salt the same PIN has the same hash on every device, so a table of a million hashes cracks any of them at once. Use Argon2id with a random per-user salt. The parameters below are OWASP's minimum for Argon2id: 19 MiB of memory, 2 iterations, 1 degree of parallelism.

```yaml
dependencies:
  cryptography: ^2.9.0
  flutter_secure_storage: ^11.2.0
  pinput: ^7.0.0
```

Each stored record carries its salt and parameters, so you can raise the cost later without breaking existing PINs. `Isolate.run` keeps the hashing off the UI thread: it takes real memory and time on purpose.

The target for an unlock is about half a second for one check, timed in a release build on the slowest phone you support, with one second as the ceiling, the limit OWASP gives for any password hash. In a release-mode build on an Apple M5 Max one hash took 193 ms, and a phone is slower. If a check finishes well under the target, raise `memory`. Never go below 19 MiB at 2 iterations to save time: OWASP's other rows, such as 12 MiB at 3 iterations, trade memory for CPU, so they spend more time rather than less.

```dart
// lib/features/auth/data/pin_hasher.dart
import 'dart:convert';
import 'dart:isolate';
import 'dart:math';

import 'package:cryptography/cryptography.dart';

final class PinRecord {
  const PinRecord({
    required this.salt,
    required this.hash,
    this.memory = 19456,
    this.iterations = 2,
  });

  factory PinRecord.fromJson(Map<String, dynamic> json) => PinRecord(
        salt: base64.decode(json['salt'] as String),
        hash: base64.decode(json['hash'] as String),
        memory: json['memory'] as int,
        iterations: json['iterations'] as int,
      );

  final List<int> salt;
  final List<int> hash;
  final int memory;
  final int iterations;

  Map<String, Object> toJson() => {
        'salt': base64.encode(salt),
        'hash': base64.encode(hash),
        'memory': memory,
        'iterations': iterations,
      };
}

List<int> randomSalt() {
  final random = Random.secure();
  return List<int>.generate(16, (_) => random.nextInt(256));
}

Future<List<int>> derivePinHash(
  String pin,
  List<int> salt, {
  int memory = 19456,
  int iterations = 2,
}) {
  return Isolate.run(() async {
    final argon2id = Argon2id(
      parallelism: 1,
      memory: memory,
      iterations: iterations,
      hashLength: 32,
    );
    final key = await argon2id.deriveKeyFromPassword(
      password: pin,
      nonce: salt,
    );
    return key.extractBytes();
  });
}

/// Compares every byte, so timing doesn't reveal the first mismatch.
bool constantTimeEquals(List<int> a, List<int> b) {
  if (a.length != b.length) return false;
  var difference = 0;
  for (var i = 0; i < a.length; i++) {
    difference |= a[i] ^ b[i];
  }
  return difference == 0;
}
```

## Enforce the PIN Attempt Limit

The attempt limit has to live in secure storage and increment before the comparison. A counter held in memory resets on every restart, and one written after the check can be dodged by killing the app the moment a wrong PIN is entered. Calls run one at a time, so two fast submissions can't race on the counter. The fifth failure deletes the PIN, so the user must sign in with their primary method and set a new one. Only a correct PIN resets the count.

```dart
// lib/features/auth/data/pin_store.dart
import 'dart:convert';

import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:yourapp/features/auth/data/pin_hasher.dart';

sealed class PinResult {
  const PinResult();
}

final class PinAccepted extends PinResult {
  const PinAccepted();
}

final class PinRejected extends PinResult {
  const PinRejected(this.attemptsLeft);

  final int attemptsLeft;
}

/// The PIN is gone: send the user through full sign-in.
final class PinLockedOut extends PinResult {
  const PinLockedOut();
}

class PinStore {
  PinStore(this._storage);

  final FlutterSecureStorage _storage;

  static const maxAttempts = 5;
  static const _recordKey = 'app_pin_record';
  static const _attemptsKey = 'app_pin_attempts';

  Future<void> _queue = Future.value();

  Future<T> _serial<T>(Future<T> Function() task) {
    final result = _queue.then((_) => task());
    _queue = result.then<void>((_) {}, onError: (Object _) {});
    return result;
  }

  Future<bool> hasPin() => _storage.containsKey(key: _recordKey);

  Future<void> setPin(String pin) async {
    final salt = randomSalt();
    final record = PinRecord(salt: salt, hash: await derivePinHash(pin, salt));
    await _storage.write(key: _recordKey, value: jsonEncode(record.toJson()));
    await _storage.delete(key: _attemptsKey);
  }

  Future<PinResult> verify(String pin) => _serial(() async {
        final raw = await _storage.read(key: _recordKey);
        if (raw == null) return const PinLockedOut();

        final count = await _storage.read(key: _attemptsKey);
        final used = int.parse(count ?? '0') + 1;
        await _storage.write(key: _attemptsKey, value: '$used');

        final record = PinRecord.fromJson(
          jsonDecode(raw) as Map<String, dynamic>,
        );
        final hash = await derivePinHash(
          pin,
          record.salt,
          memory: record.memory,
          iterations: record.iterations,
        );
        if (constantTimeEquals(hash, record.hash)) {
          await _storage.delete(key: _attemptsKey);
          return const PinAccepted();
        }
        if (used >= maxAttempts) {
          await clearPin();
          return const PinLockedOut();
        }
        return PinRejected(maxAttempts - used);
      });

  Future<void> clearPin() async {
    await _storage.delete(key: _recordKey);
    await _storage.delete(key: _attemptsKey);
  }
}
```

## PIN Entry Screen with pinput

`pinput` 7 imports `material_ui` and needs Flutter 3.44 or later; a project still on `package:flutter/material.dart` stays on 6.x.

```dart
// lib/features/auth/presentation/pin_entry_page.dart
import 'package:material_ui/material_ui.dart';
import 'package:pinput/pinput.dart';
import 'package:yourapp/features/auth/data/pin_store.dart';

class PinEntryPage extends StatefulWidget {
  const PinEntryPage({
    super.key,
    required this.store,
    required this.onUnlocked,
    required this.onLockedOut,
    this.onUseBiometric,
  });

  final PinStore store;
  final VoidCallback onUnlocked;
  final VoidCallback onLockedOut;

  /// Null when biometrics aren't available.
  final VoidCallback? onUseBiometric;

  @override
  State<PinEntryPage> createState() => _PinEntryPageState();
}

class _PinEntryPageState extends State<PinEntryPage> {
  final _controller = TextEditingController();
  String? _error;
  bool _checking = false;

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Future<void> _verify(String pin) async {
    setState(() => _checking = true);
    final result = await widget.store.verify(pin);
    if (!mounted) return;
    setState(() => _checking = false);
    switch (result) {
      case PinAccepted():
        widget.onUnlocked();
      case PinLockedOut():
        widget.onLockedOut();
      case PinRejected(:final attemptsLeft):
        _controller.clear();
        setState(() => _error = 'Wrong PIN. $attemptsLeft attempts left.');
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final cell = PinTheme(
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

    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Enter your PIN', style: theme.textTheme.headlineMedium),
              const SizedBox(height: 32),
              Pinput(
                length: 6,
                controller: _controller,
                enabled: !_checking,
                defaultPinTheme: cell,
                focusedPinTheme: cell.copyBorderWith(
                  border: Border.all(color: theme.colorScheme.primary, width: 2),
                ),
                errorPinTheme: cell.copyBorderWith(
                  border: Border.all(color: theme.colorScheme.error),
                ),
                errorText: _error,
                forceErrorState: _error != null,
                obscureText: true,
                obscuringCharacter: '●',
                hapticFeedbackType: HapticFeedbackType.lightImpact,
                onCompleted: _verify,
              ),
              if (_checking)
                const Padding(
                  padding: EdgeInsets.only(top: 24),
                  child: CircularProgressIndicator(),
                ),
              if (widget.onUseBiometric case final useBiometric?)
                TextButton(
                  onPressed: useBiometric,
                  child: const Text('Use Face ID or fingerprint'),
                ),
              TextButton(
                onPressed: widget.onLockedOut,
                child: const Text('Forgot PIN?'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## Biometric to PIN Fallback Hierarchy

The order is biometric, then PIN, then full sign-in. A biometric that's cancelled or locked out falls through to the PIN, and a device with neither goes straight to the provider.

```dart
// lib/features/auth/domain/local_unlock.dart
import 'package:yourapp/features/auth/data/biometric_data_source.dart';
import 'package:yourapp/features/auth/data/pin_store.dart';

enum UnlockStep { unlocked, askForPin, fullSignIn }

Future<UnlockStep> startUnlock(BiometricDataSource biometric, PinStore pin) async {
  final capability = await biometric.checkCapability();
  if (capability == BiometricCapability.available) {
    try {
      final passed = await biometric.authenticate(
        reason: 'Verify your identity to continue',
      );
      if (passed) return UnlockStep.unlocked;
    } on BiometricFailure {
      // Locked out or cancelled: fall through to the PIN.
    }
  }
  return await pin.hasPin() ? UnlockStep.askForPin : UnlockStep.fullSignIn;
}
```

## Tests for the PIN Store

Run these with `FlutterSecureStorage.setMockInitialValues({})`.

```dart
// test/features/auth/pin_store_test.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/features/auth/data/pin_hasher.dart';
import 'package:yourapp/features/auth/data/pin_store.dart';

void main() {
  late PinStore store;

  setUp(() {
    FlutterSecureStorage.setMockInitialValues({});
    store = PinStore(const FlutterSecureStorage());
  });

  test('the same PIN hashes differently under two salts', () async {
    final a = await derivePinHash('123456', randomSalt());
    final b = await derivePinHash('123456', randomSalt());
    expect(constantTimeEquals(a, b), isFalse);
  });

  test('the attempt limit survives a restart and deletes the PIN', () async {
    await store.setPin('123456');
    for (var attempt = 1; attempt < PinStore.maxAttempts; attempt++) {
      final restarted = PinStore(const FlutterSecureStorage());
      expect(await restarted.verify('000000'), isA<PinRejected>());
    }
    expect(await store.verify('000000'), isA<PinLockedOut>());
    expect(await store.hasPin(), isFalse);
    expect(await store.verify('123456'), isA<PinLockedOut>());
  });

  test('parallel attempts cannot slip past the counter', () async {
    await store.setPin('123456');
    final results = await Future.wait(
      List.generate(PinStore.maxAttempts + 2, (_) => store.verify('000000')),
    );
    expect(results.whereType<PinLockedOut>(), hasLength(3));
  });

  test('a correct PIN resets the counter', () async {
    await store.setPin('123456');
    await store.verify('000000');
    expect(await store.verify('123456'), isA<PinAccepted>());
    expect(
      await store.verify('000000'),
      isA<PinRejected>().having((r) => r.attemptsLeft, 'left', 4),
    );
  });
}
```
