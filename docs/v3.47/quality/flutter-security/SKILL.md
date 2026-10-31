---
name: flutter-security
description: Hardens a Flutter app before release against the OWASP Mobile Top 10 (2024) - certificate pinning with dio, envied build-time config, --obfuscate with symbols, flutter_secure_storage, root and jailbreak signals, screenshot blocking, SQLCipher databases, R8 rules and a release checklist. Use when preparing a production release, pinning a certificate, storing tokens, encrypting a local database, or when someone asks is this app secure.
metadata:
  tags:
  - flutter
  - security
  - owasp
  - obfuscation
  - certificate-pinning
  - secure-storage
---
# Flutter Security

Hardening work for a Flutter app that is about to meet real users, built around the OWASP Mobile Top 10 (2024). The app runs on a device the attacker may own, so nothing here makes a secret safe inside the binary. What these controls do is keep tokens out of plain storage, keep traffic on your server, and raise the cost of tampering.

Run it on project setup, after a new API host or sensitive data type, and as a checklist on every release candidate.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(dio|envied|envied_generator|build_runner|flutter_secure_storage|screen_protector|flutter_jailbreak_detection|drift|sqlite3):" pubspec.yaml
dart pub outdated --no-dev-dependencies
```

These pins were read from pub.dev on 2026-10-30 against Flutter 3.47.9 and Dart 3.13.7, and the Dart examples were analyzed against them. `build_runner: ^2.16.1` sits below the latest 2.17.1 on purpose: 2.17.1 needs `build` 5, which `envied_generator`, `hive_ce_generator`, `injectable_generator`, `isar_community_generator` and `riverpod_generator` don't accept yet, and 2.16.2 needs `analyzer` 14.3, which `injectable_generator` 3.1.3 can't take. In a project on `package:flutter/material.dart`, swap the `material_ui` import in these examples; `dart fix --apply --code=migrate_design_widgets` does it for a whole project. Everything here was compiled, not run on a device, so prove the Keystore, Keychain and screen protection steps on real hardware. The examples also import `material_ui: ^1.9.0`, `path: ^1.9.1`, `path_provider: ^2.1.6`.

## Which Control Stops Which Threat

| OWASP 2024 risk | Control | Why the tempting option loses |
|---|---|---|
| M1 Improper Credential Usage | Keep real secrets on your server; ship only public identifiers | `envied` obfuscation slows a reader down, and its own README says the value is still recoverable |
| M2 Inadequate Supply Chain Security | Commit `pubspec.lock`, use `--enforce-lockfile`, read changelogs on upgrade | Floating versions change what you ship without a review |
| M3 Insecure Authentication/Authorization | Check permissions on the server | A client check is a patchable `if` |
| M4 Insufficient Input/Output Validation | Validate on the server, and treat deep links as hostile | A route guard alone can't protect data |
| M5 Insecure Communication | TLS everywhere, pinning for high-value APIs | `badCertificateCallback` never pins: it runs only for rejected certificates |
| M6 Inadequate Privacy Controls | Collect less, mask screens, keep PII out of logs | Encrypting data you never needed still leaks it through your own API |
| M7 Insufficient Binary Protections | `--obfuscate`, R8, integrity signals on the server | Flutter's docs say obfuscation isn't security: it only renames symbols |
| M8 Security Misconfiguration | Block cleartext, drop unused permissions | Debug settings leak through build flavors |
| M9 Insecure Data Storage | Keychain and Keystore through `flutter_secure_storage`, SQLCipher for databases | `SharedPreferences` and Hive are plain files |
| M10 Insufficient Cryptography | Use platform crypto and vetted libraries | Hand-rolled encryption and unsalted hashes fail quietly |

## Certificate Pinning with dio

Certificate pinning makes the client refuse a TLS peer you didn't choose, even one signed by a CA the device trusts. Pin only what you can rotate on your own schedule: a wrong pin takes the whole installed base offline until an update lands.

```yaml
dependencies:
  dio: ^5.11.1
  crypto: ^3.0.8
```

`IOHttpClientAdapter.validateCertificate` is the hook that pins. dio calls it after the TLS checks pass, on every response, with the leaf certificate. `HttpClient.badCertificateCallback` can't pin, because it fires only for a certificate the platform already rejected, so a valid attacker certificate never reaches it. `NativeAdapter` from `native_dio_adapter` has neither hook: a pinned client uses `dart:io`, and gives up the native stack's system VPN and proxy support.

```dart
// lib/core/network/pinned_dio.dart
import 'dart:io';
import 'dart:typed_data';

import 'package:crypto/crypto.dart';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

/// SHA-256 of the accepted leaf certificates, lowercase hex.
/// Always two entries: the live certificate and the next one.
const apiPins = <String>{
  '4f1c0b7a6d2e9f835c10ab47e6d2318f9ac7b05de14f6a2839bd70c1e5a2f64d', // live
  'a93e7d15c04b6f28e1d97a3c5f08b24d6e13c97fa0528bd4316ce7f09ab1d852', // next
};

/// Pure, so it can be tested without a network.
bool matchesPin(Uint8List? der, Set<String> pins) {
  if (der == null) return false; // No certificate means no proof.
  return pins.contains(sha256.convert(der).toString());
}

/// [issuerCertificate] is your CA or intermediate. With the system roots off,
/// a stranger's certificate fails during the handshake, before any header or
/// token leaves the device.
Dio createPinnedDio({
  required String baseUrl,
  required Uint8List issuerCertificate,
  Set<String> pins = apiPins,
}) {
  final dio = Dio(
    BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 20),
    ),
  );
  dio.httpClientAdapter = IOHttpClientAdapter(
    createHttpClient: () {
      final context = SecurityContext()
        ..setTrustedCertificatesBytes(issuerCertificate);
      return HttpClient(context: context)
        ..badCertificateCallback = (_, _, _) => false;
    },
    validateCertificate: (certificate, host, port) =>
        matchesPin(certificate?.der, pins),
  );
  return dio;
}
```

The `SecurityContext` line matters because of when dio checks. `validateCertificate` runs after the response arrives, so on its own it blocks the reply but not the request: a MITM has already read the `Authorization` header. Restricting the trusted roots stops that at the handshake. `SecurityContext()` starts with `withTrustedRoots: false`. The Dart SDK docs ask for a single DER certificate on iOS and PEM or PKCS12 elsewhere, so ship both assets and load by `Platform.isIOS`. We analyzed this code, but didn't run it on a device.

Fingerprints here are the SHA-256 of the certificate's DER bytes:

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -outform DER | openssl dgst -sha256 -r | cut -d' ' -f1
```

**Rotate in this order:** add the next certificate's pin, release, wait until about 95% of active users have updated, install the new certificate on the server, then remove the old pin in the next release. Skipping the wait locks out everyone left on the old build.

## Build-Time Config with envied

`envied` generates a Dart class from a `.env` file at build time and can XOR the values. Use it for per-environment values you'd rather not commit, such as URLs and public keys. Its README states the limit: this makes values harder to read, and doesn't make them secret. A key that spends money or reads other users' data belongs on your server.

```yaml
dependencies:
  envied: ^1.3.10

dev_dependencies:
  envied_generator: ^1.3.10
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
```

```dart
// lib/core/env/env.dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: '.env', obfuscate: true)
abstract class Env {
  @EnviedField(varName: 'SUPABASE_URL')
  static final String supabaseUrl = _Env.supabaseUrl;

  @EnviedField(varName: 'SUPABASE_PUBLISHABLE_KEY')
  static final String supabasePublishableKey = _Env.supabasePublishableKey;

  @EnviedField(varName: 'API_BASE_URL')
  static final String apiBaseUrl = _Env.apiBaseUrl;
}
```

```gitignore
# .gitignore: the source values and the generated copy of them
.env
*.env
lib/core/env/env.g.dart
```

Run `dart run build_runner build`. The generated `env.g.dart` holds XOR-obfuscated values, so plain `strings` output won't show them.

## Binary Obfuscation for Release Builds

Obfuscation replaces Dart class and function names with short symbols. It doesn't encrypt resources or string literals, and Flutter's docs say plainly that it isn't a security measure. It needs a release build and doesn't apply to web.

```bash
flutter build appbundle --release --obfuscate --split-debug-info=build/debug-symbols/android
flutter build ipa --release --obfuscate --split-debug-info=build/debug-symbols/ios
```

Keep the symbols directory for every shipped build: without it, a stack trace stays unreadable. Store it as a CI artifact named by build number, and never commit it. To read a trace by hand, run `flutter symbolize -i <trace> -d <symbols-file>`. For Crashlytics on Android, upload the directory with Firebase CLI 11.9.0 or later, using the `mobilesdk_app_id` value from `google-services.json`:

```bash
firebase crashlytics:symbols:upload --app=FIREBASE_APP_ID build/debug-symbols/android
```

On Apple platforms, `flutterfire configure` tries to add a build phase that uploads dSYMs. Check that it's there. Code that reads `runtimeType.toString()` breaks once names change; enum names aren't obfuscated.

## Secure Storage with flutter_secure_storage

`flutter_secure_storage` keeps values in the iOS Keychain and in an Android store whose key is wrapped by the Keystore: AES-GCM data with an RSA-OAEP-wrapped key by default.

```yaml
dependencies:
  flutter_secure_storage: ^11.2.0
```

```dart
// lib/core/storage/secure_storage.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorage {
  const SecureStorage();

  static const FlutterSecureStorage _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(),
    iOptions: IOSOptions(
      // Readable in the background after first unlock; never in a backup.
      accessibility: KeychainAccessibility.first_unlock_this_device,
    ),
  );

  Future<void> write(String key, String value) =>
      _storage.write(key: key, value: value);

  Future<String?> read(String key) => _storage.read(key: key);

  Future<void> delete(String key) => _storage.delete(key: key);

  Future<void> deleteAll() => _storage.deleteAll();
}
```

| Data | Store in |
|---|---|
| Auth and refresh tokens, encryption keys | `flutter_secure_storage` |
| User PII (email, name) | `flutter_secure_storage` or an encrypted database |
| App settings, feature flags | Hive or `SharedPreferences` |
| Analytics opt-out | `SharedPreferences` |

Version 11 removed `encryptedSharedPreferences` and `sharedPreferencesName` (use `storageNamespace`) and raised `minSdk` to 24. Data written before version 10 must pass through 10 first, or it can't be read; `checkUpgradeStatus()` reports that case. `AndroidOptions` defaults to `resetOnError: true`, so a decrypt failure wipes the store: treat every read as possibly empty.

For a value that needs a biometric check on each use, `AndroidOptions.biometric(enforceBiometrics: true, requireBiometricsPerOperation: true)` and `IOSOptions(accessControlFlags: [...])` gate the key in the OS, so a patched Dart `if` can't skip it. Leave `requireBiometricsPerOperation` out and Android asks once: since 11.2.0 the unlocked app key stays in memory until the app restarts, and later reads don't prompt. `references/android-hardening.md` has that setup.
## Root and Jailbreak Detection

`flutter_jailbreak_detection` gives two async signals: `jailbroken` and `developerMode`. Both are hints, and a determined attacker can hide from them, so the decision belongs on your server alongside App Check or Play Integrity. Developer options alone mean a developer, not an attacker.

```yaml
dependencies:
  flutter_jailbreak_detection: ^1.10.0
```

```dart
// lib/core/security/device_integrity.dart
import 'package:flutter/services.dart';
import 'package:flutter_jailbreak_detection/flutter_jailbreak_detection.dart';

final class DeviceIntegrity {
  const DeviceIntegrity({required this.compromised, required this.developerMode});

  final bool compromised;
  final bool developerMode;
}

Future<DeviceIntegrity> checkDeviceIntegrity() async {
  try {
    final results = await Future.wait([
      FlutterJailbreakDetection.jailbroken,
      FlutterJailbreakDetection.developerMode,
    ]);
    return DeviceIntegrity(compromised: results[0], developerMode: results[1]);
  } on PlatformException {
    // A failed check proves nothing, so it never blocks a real user.
    return const DeviceIntegrity(compromised: false, developerMode: false);
  } on MissingPluginException {
    return const DeviceIntegrity(compromised: false, developerMode: false);
  }
}
```

Choose the response per app. Banking and payments can refuse to run. The default is softer: send the signal to the server, and switch off biometric unlock and offline storage of sensitive data on that device.

## Screen Privacy with screen_protector

`screen_protector` covers both platforms. `flutter_windowmanager` was last published in 2021 and its SDK bound stops below Dart 3, so don't use it.

```yaml
dependencies:
  screen_protector: ^1.5.3
```

```dart
// lib/features/payment/presentation/pages/payment_page.dart
import 'package:material_ui/material_ui.dart';
import 'package:screen_protector/screen_protector.dart';

class PaymentPage extends StatefulWidget {
  const PaymentPage({super.key, required this.child});

  final Widget child;

  @override
  State<PaymentPage> createState() => _PaymentPageState();
}

class _PaymentPageState extends State<PaymentPage> {
  @override
  void initState() {
    super.initState();
    ScreenProtector.preventScreenshotOn();
    ScreenProtector.protectDataLeakageWithBlur(); // iOS app switcher only.
  }

  @override
  void dispose() {
    ScreenProtector.preventScreenshotOff();
    ScreenProtector.protectDataLeakageWithBlurOff();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

On Android, `preventScreenshotOn` sets `FLAG_SECURE`, which also keeps the screen out of recordings and the recents thumbnail. iOS is different: we found no public Apple API for blocking a screenshot, and `preventScreenshotOn` there comes from the third-party `ScreenProtectorKit`, so retest on each major iOS release. The `protectDataLeakageWith*` calls cover the separate iOS case of the app-switcher snapshot. Apply both to PIN entry, payment flows, account details and any screen with a full account number.

`references/screen-privacy.md` has a Dart-only app-switcher cover for apps that avoid the plugin.

## Android and Database Hardening

| Task | Where |
|---|---|
| Block cleartext, OS-level pins, debug overrides, R8 keep rules, `excludeFromRecents`, biometric-bound keys | `references/android-hardening.md` |
| Encrypt a Drift database with `source: sqlcipher`, the key, the startup check, one-time migration | `references/encrypted-database.md` |

## Verify

Prove pinning with a test, then with a real proxy. These cases cover what a matcher must refuse:

```dart
// test/core/network/pinned_dio_test.dart
import 'dart:typed_data';

import 'package:crypto/crypto.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/core/network/pinned_dio.dart';

void main() {
  final good = Uint8List.fromList([1, 2, 3]);
  final pins = {sha256.convert(good).toString()};

  test('accepts a pinned certificate', () {
    expect(matchesPin(good, pins), isTrue);
  });

  test('refuses an unknown certificate', () {
    expect(matchesPin(Uint8List.fromList([9]), pins), isFalse);
  });

  test('refuses a missing certificate', () {
    expect(matchesPin(null, pins), isFalse);
  });

  test('refuses everything when no pin is configured', () {
    expect(matchesPin(good, const {}), isFalse);
  });
}
```

Then run the release build on a test device behind mitmproxy or Charles with the proxy CA installed. Requests must fail with `DioExceptionType.badCertificate` or a `HandshakeException`, and the proxy log must show no request headers.

```bash
git log --all --oneline -- .env                   # expect nothing
grep -rnE "sk_live_|service_role|http://" lib/    # expect nothing but localhost in debug code
unzip -p build/app/outputs/flutter-apk/app-release.apk lib/arm64-v8a/libapp.so | strings | grep -c "MyClassName"   # expect 0
```

## Gotchas

1. **The pin never rejects anything.** Symptom: a proxy reads traffic although the code "pins". Cause: the check is in `badCertificateCallback`, which runs only for rejected certificates. Fix: use `validateCertificate`, plus a restricted `SecurityContext`.
2. **`The named parameter 'createHttpClient' isn't defined`.** Symptom: a compile error on `NativeAdapter`. Cause: that adapter has no such parameter and no pinning hook. Fix: use `IOHttpClientAdapter` for pinned hosts.
3. **Every user is offline after a renewal.** Symptom: `DioExceptionType.badCertificate` for everyone after a certificate change. Cause: no backup pin, or the server rotated first. Fix: ship both pins, wait for adoption, then rotate.
4. **`because every version of envied_generator ... build` fails to resolve.** Symptom: `version solving failed` naming `build`. Cause: `build_runner` 2.17.1 needs `build` 5. Fix: use `build_runner: ^2.16.1`.
5. **Secure storage is empty after an update.** Symptom: users are signed out once. Cause: `resetOnError`, or a jump across versions 9 to 11. Fix: go through 10, and treat an empty read as signed out.
6. **A token works after a reinstall on iOS.** Symptom: a fresh install is already signed in. Cause: Keychain items survive app removal. Fix: on first launch, check a `SharedPreferences` flag and clear the Keychain.
7. **The crash report shows only symbols.** Symptom: unreadable traces after `--obfuscate`. Cause: symbols weren't uploaded. Fix: upload them in the release pipeline, per build.

## Anti-Patterns

- Treating `envied` as a vault. Put a server between the app and any real secret.
- Forcing an exit on a root signal in a consumer app. It punishes enthusiasts and barely slows attackers.
- `assert` for a security check. Asserts are stripped from release builds, which is where the check matters.
- Logging tokens or full request bodies, even in debug helpers that can reach a release build.
- Pinning a third-party API whose certificates you don't control.

## Pre-Merge Checklist

- [ ] Release build uses `--obfuscate --split-debug-info`, and symbols are archived by build version
- [ ] `.env` and `env.g.dart` are ignored, and `git log` shows neither was ever committed
- [ ] Pins are current, a backup pin is set, and the certificate expiry date is on a calendar
- [ ] No private key in `lib/`, no `http://` endpoint in production code
- [ ] Tokens and PII live only in secure storage or an encrypted database
- [ ] A root or jailbreak policy exists and reaches the server
- [ ] R8 stays on (no `--no-shrink`) and keep rules are tested on the release build
- [ ] `network_security_config.xml` is deployed and referenced in `AndroidManifest.xml`
- [ ] iOS has no `NSAllowsArbitraryLoads: true`
- [ ] Minimum SDKs are iOS 15 and Android API 24
- [ ] Unused permissions are gone from both manifests
- [ ] PIN, payment and account screens are protected on both platforms
- [ ] Database encryption is on, and any plaintext file was migrated and deleted
- [ ] `excludeFromRecents` is set for sensitive activities where the app needs it
- [ ] Biometric-bound keys were tested on Android API 30+ and on Face ID and Touch ID hardware

## See Also

- `flutter-auth` for sign-in, token lifecycle and PIN lock
- `flutter-device-intelligence` for App Check and device context
- `flutter-network` for general Dio setup and interceptors
- `flutter-database` for Drift beyond encryption
