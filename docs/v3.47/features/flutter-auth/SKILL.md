---
name: flutter-auth
description: Builds sign-in for a Flutter app - Firebase Auth or Supabase Auth, Google and Apple sign-in, biometrics with local_auth, token storage, an explicit auth state, go_router guards and complete logout, plus PIN lock, MFA, TOTP and magic links in references. Use when adding login or registration, Face ID or fingerprint, Sign in with Apple, a login redirect, session handling or logout, or when App Review rejects a login under guideline 4.8.
metadata:
  tags:
  - flutter
  - auth
  - biometrics
  - firebase
  - supabase
  - oauth
---
# Flutter Auth

The identity layer for a production Flutter app: who the user is, how the session survives, and what happens at each edge of signing in and out. Auth bugs land on the worst day, with a user locked out or a token left behind, so every state here is explicit and each path has a test.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(firebase_auth|firebase_core|supabase_flutter|google_sign_in|sign_in_with_apple|local_auth|flutter_secure_storage|go_router):" pubspec.yaml
dart pub outdated --no-dev-dependencies
```

Pins were read from pub.dev on 2026-10-30 for Flutter 3.47.9 and Dart 3.13.7, and the examples were analyzed against them. `build_runner: ^2.16.1` sits below the latest 2.17.1 on purpose: 2.17.1 needs `build` 5, which `envied_generator`, `hive_ce_generator`, `injectable_generator`, `isar_community_generator` and `riverpod_generator` don't accept yet, and 2.16.2 needs `analyzer` 14.3, which `injectable_generator` 3.1.3 can't take. On `package:flutter/material.dart`, swap the `material_ui` import in these examples, or run `dart fix --apply --code=migrate_design_widgets`. Everything here was compiled, not run on a device, so try each sign-in and the biometric prompt on real hardware. The examples also import `dio: ^5.11.1`, `firebase_analytics: ^13.0.1`, `google_sign_in: ^7.2.0`, `material_ui: ^1.9.0`, `qr_flutter: ^4.1.0`, `riverpod_annotation: ^4.0.7`.

## Choosing an Auth Approach

| Need | Choose | Why the others lose here |
|---|---|---|
| Firebase is already your backend | Firebase Auth | Supabase would mean a second user table to sync |
| Postgres with row level security | Supabase Auth | Firebase tokens need custom wiring for RLS |
| Your own API owns users | OAuth or your own endpoints | Provider SDKs add a dependency you don't use |
| Quick return to a signed-in app | `local_auth`, then a stored session | A password prompt on each launch loses users |
| A secret that must stay locked without a biometric | Keystore or Keychain gating in `flutter_secure_storage` | A `local_auth` boolean is an `if` an attacker can patch |

Whichever you pick, the provider decides how long a session lives. Treat a signed-out event as normal, because a token can be revoked at any time.

## Auth State Machine

Define the state as a sealed class before writing any sign-in code, so the compiler forces every screen to handle every case. `AuthState` would collide with the class of that name in `supabase_flutter`, so this skill uses `AppAuthState`.

```dart
// lib/features/auth/domain/app_auth_state.dart
final class AppUser {
  const AppUser({required this.id, this.email});

  final String id;
  final String? email;
}

sealed class AppAuthState {
  const AppAuthState();
}

final class AuthLoading extends AppAuthState {
  const AuthLoading();
}

final class AuthSignedOut extends AppAuthState {
  const AuthSignedOut();
}

final class AuthSignedIn extends AppAuthState {
  const AuthSignedIn(this.user);

  final AppUser user;
}

/// A restored session, with the app locked until a biometric or PIN check.
final class AuthRequiresBiometric extends AppAuthState {
  const AuthRequiresBiometric(this.user);

  final AppUser user;
}

/// Signed in with a password, but the address isn't confirmed yet.
final class AuthRequiresEmailVerification extends AppAuthState {
  const AuthRequiresEmailVerification(this.email);

  final String? email;
}

final class AuthFailed extends AppAuthState {
  const AuthFailed(this.message);

  final String message;
}
```

## Auth-Guarded go_router Navigation

Write the guard as a pure function, so it's testable without a widget tree. `GoRouter` only re-runs `redirect` when something tells it to, so `refreshListenable` has to fire on each auth change.

```dart
// lib/core/router/auth_redirect.dart
import 'package:yourapp/features/auth/domain/app_auth_state.dart';

String? authRedirect(AppAuthState auth, String location) {
  final onAuthRoute = location == '/auth' || location.startsWith('/auth/');
  return switch (auth) {
    AuthLoading() => location == '/splash' ? null : '/splash',
    AuthSignedIn() => onAuthRoute || location == '/splash' ? '/' : null,
    AuthRequiresBiometric() => location == '/auth/unlock' ? null : '/auth/unlock',
    AuthRequiresEmailVerification() =>
      location == '/auth/verify-email' ? null : '/auth/verify-email',
    AuthSignedOut() || AuthFailed() => onAuthRoute ? null : '/auth/login',
  };
}
```

## Biometric Authentication with local_auth

`local_auth` 3 removed `AuthenticationOptions` and shows no error dialogs of its own. It returns a boolean, so use it to reopen an app, not to protect a server action. `references/provider-data-sources.md` has the complete data source, with capability checks and error mapping.

```yaml
dependencies:
  local_auth: ^3.0.2
```

```xml
<!-- ios/Runner/Info.plist -->
<key>NSFaceIDUsageDescription</key>
<string>Use Face ID to securely sign in to your account</string>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.USE_BIOMETRIC"/>
```

Android needs two more changes, and the Flutter template has neither. `MainActivity` must extend `FlutterFragmentActivity`, and `LaunchTheme` must have a `Theme.AppCompat` parent. Without the first, the prompt never shows; the second prevents crashes on Android 8 and below. `references/provider-data-sources.md` has both files.

## Firebase Email Sign-In and invalid-credential

Firebase projects created since September 15, 2023 have email enumeration protection on by default. Sign-in then answers `invalid-credential` for both a wrong password and an unknown address, so `user-not-found` and `wrong-password` no longer appear there. Older projects can still send them, so map all three to one message.

```yaml
dependencies:
  firebase_auth: ^7.0.1
  firebase_core: ^5.1.0
```

Map every code in one function, so screens see a single error type (`references/provider-data-sources.md`). Firebase needs `ios/Runner/GoogleService-Info.plist` from the console, plus `android/app/google-services.json` and a registered SHA-1 fingerprint on Android.

## Google Sign-In with GoogleSignIn.instance

`google_sign_in` 7 replaced the 6.x API, so older snippets no longer compile. `GoogleSignIn` is a singleton reached through `GoogleSignIn.instance`, and the plugin no longer tracks a current user: listen to `authenticationEvents` instead. `signIn` became `authenticate`, `signInSilently` became `attemptLightweightAuthentication`, and `clearAuthCache` became `clearAuthorizationToken`. Authentication and authorization are now separate: ask for an access token with `authorizationClient.authorizationForScopes(scopes)`, fall back to `authorizeScopes(scopes)` when it returns null, and call `authorizeServer` for a `serverAuthCode`. `authenticate` isn't supported on web, so check `supportsAuthenticate()` and render the button from `google_sign_in_web`.

`initialize` must run exactly once, before any other call, and Android needs `serverClientId`, the web client ID from the Firebase console. `account.authentication` is synchronous in 7.x and carries only the ID token. `references/provider-data-sources.md` has the data source.

## Apple Sign In and App Store Guideline 4.8

Guideline 4.8 doesn't name Sign in with Apple. An app that uses a third-party login for the user's primary account must also offer an equivalent login service that limits data collection to name and email, lets users keep their email private, and doesn't collect interactions for advertising without consent. Sign in with Apple meets all three, which is why most apps add it, but any service with those features qualifies.

No second service is needed when the app uses only your company's own accounts, is an education, enterprise or business app using an existing account of that kind, uses a government or industry citizen ID, is a client for one particular third-party service, or is an alternative marketplace with its own login.

```yaml
dependencies:
  sign_in_with_apple: ^8.2.0
  crypto: ^3.0.8
```

Enable the capability in Xcode under Signing & Capabilities, and turn on Apple as a provider. The Services ID and registered domain are needed only for Android and web flows. Send Apple a SHA-256 hash of a random nonce, then give Firebase the raw value through `AppleAuthProvider.credentialWithIDToken`, as in `references/provider-data-sources.md`. Apple sends the name and email only on the first authorization, so store them at that moment.

Guideline 5.1.1(v) adds two duties. An app that supports account creation must also offer account deletion inside the app, and the app must give users a way to revoke social network credentials.

## Supabase Auth with a Secure Session Store

`supabase_flutter` handles token refresh on its own and uses the PKCE flow by default. Its default session store is `SharedPreferences`, a plain file, so replace it to meet the storage rule below. `anonKey` is deprecated: pass `publishableKey`.

```yaml
dependencies:
  supabase_flutter: ^2.18.3
  flutter_secure_storage: ^11.2.0
```

`references/provider-data-sources.md` has a `LocalStorage` implementation over `flutter_secure_storage` and the data source. Listen to `onAuthStateChange` for `AuthChangeEvent.signedOut`, which also fires when a session ends without a user action. `references/email-and-magic-link.md` covers the deep link setup the redirect needs.

## Secure Token Storage

All tokens, session data and credentials go in `flutter_secure_storage`, never `SharedPreferences`. The Firebase SDK manages its own tokens, so this class is for your own API.

```dart
// lib/core/storage/token_store.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStore {
  const TokenStore();

  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(), // RSA-OAEP key wrapping + AES-GCM by default.
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
    ),
  );

  static const _accessKey = 'access_token';
  static const _refreshKey = 'refresh_token';

  Future<void> save({required String access, required String refresh}) async {
    await _storage.write(key: _accessKey, value: access);
    await _storage.write(key: _refreshKey, value: refresh);
  }

  Future<String?> get accessToken => _storage.read(key: _accessKey);

  Future<void> clear() async {
    await _storage.delete(key: _accessKey);
    await _storage.delete(key: _refreshKey);
  }
}
```

## Complete Logout

A complete logout signs out of the provider, clears stored tokens, cancels live subscriptions, clears the user's cache and returns to sign-in, all in one operation. Local cleanup runs even when the network call fails: a user who taps Sign out on a plane must end up signed out on that device.

```dart
// lib/features/auth/presentation/auth_notifier.dart
import 'dart:async';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:yourapp/core/storage/token_store.dart';
import 'package:yourapp/features/auth/data/firebase_auth_data_source.dart';
import 'package:yourapp/features/auth/domain/app_auth_state.dart';

part 'auth_notifier.g.dart';

@Riverpod(keepAlive: true)
FirebaseAuthDataSource firebaseAuthDataSource(Ref ref) =>
    FirebaseAuthDataSource(FirebaseAuth.instance);

@Riverpod(keepAlive: true)
TokenStore tokenStore(Ref ref) => const TokenStore();

/// Overridden in main() with the person's "lock this app" setting.
@Riverpod(keepAlive: true)
bool biometricLockEnabled(Ref ref) => false;

@Riverpod(keepAlive: true)
class AuthNotifier extends _$AuthNotifier {
  bool _unlocked = false; // a password or a biometric passed in this run

  @override
  AppAuthState build() {
    final subscription = ref
        .watch(firebaseAuthDataSourceProvider)
        .authStateChanges
        .listen(_onUserChanged);
    ref.onDispose(subscription.cancel);
    return const AuthLoading();
  }

  void _onUserChanged(User? user) {
    final locked = ref.read(biometricLockEnabledProvider) && !_unlocked;
    state = switch (user) {
      null => const AuthSignedOut(),
      User(emailVerified: false, email: final email?) =>
        AuthRequiresEmailVerification(email),
      final user when locked => AuthRequiresBiometric(
        AppUser(id: user.uid, email: user.email),
      ),
      final user => AuthSignedIn(AppUser(id: user.uid, email: user.email)),
    };
  }

  /// The unlock screen calls this once the biometric or PIN check passes.
  void unlock() {
    if (state case AuthRequiresBiometric(:final user)) {
      _unlocked = true;
      state = AuthSignedIn(user);
    }
  }

  /// Leaves the state alone while the call runs: `AuthLoading` would send the
  /// router to /splash and drop the form. The form shows its own spinner.
  Future<void> signIn(String email, String password) async {
    _unlocked = true; // the password just proved who this is
    try {
      await ref.read(firebaseAuthDataSourceProvider).signIn(email, password);
    } on AppAuthException catch (e) {
      _unlocked = false;
      state = AuthFailed(e.code.name);
    }
  }

  /// `authStateChanges` stays silent when a flag changes, so the
  /// verification screen calls this after reloading the user.
  void onEmailVerified(User user) => _onUserChanged(user);

  /// [clearUserData] closes live subscriptions and empties the user's cache.
  Future<void> signOut({Future<void> Function()? clearUserData}) async {
    state = const AuthLoading();
    try {
      await ref.read(firebaseAuthDataSourceProvider).signOut();
    } on FirebaseAuthException {
      // A failed network call must not leave local data behind.
    } finally {
      await ref.read(tokenStoreProvider).clear();
      await clearUserData?.call();
      _unlocked = false;
      state = const AuthSignedOut();
    }
  }
}
```

## PIN, MFA, Email Links and Events

| Task | Where |
|---|---|
| PIN lock: salted Argon2id, attempt limit, entry UI, the biometric to PIN fallback | `references/pin-lock.md` |
| SMS codes, TOTP authenticators, backup codes, all verified on the server | `references/mfa-and-otp.md` |
| Email verification screen, Supabase magic link, deep link setup | `references/email-and-magic-link.md` |
| Analytics events for each auth step | `references/auth-analytics.md` |

## Verify

```dart
// test/core/router/auth_redirect_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/core/router/auth_redirect.dart';
import 'package:yourapp/features/auth/domain/app_auth_state.dart';

void main() {
  const user = AppUser(id: 'u1');

  test('a signed-out user cannot reach a private route', () {
    expect(authRedirect(const AuthSignedOut(), '/account'), '/auth/login');
  });

  test('a signed-in user leaves the login screen', () {
    expect(authRedirect(const AuthSignedIn(user), '/auth/login'), '/');
  });

  test('a route that only starts like /auth stays a normal route', () {
    expect(authRedirect(const AuthSignedOut(), '/authors'), '/auth/login');
    expect(authRedirect(const AuthSignedIn(user), '/authors'), isNull);
  });

  test('an unverified user stays on the verification screen', () {
    const state = AuthRequiresEmailVerification('a@example.com');
    expect(authRedirect(state, '/account'), '/auth/verify-email');
  });

  test('a locked app stays on the unlock screen', () {
    expect(authRedirect(const AuthRequiresBiometric(user), '/'), '/auth/unlock');
  });
}
```

Run these by hand on a device: a wrong password and an unknown address show the same message; sign out in airplane mode, then relaunch offline; cancel Google and Apple sheets without an error screen; deny the biometric, then use the fallback.

```bash
grep -rnE "SharedPreferences.*(token|session)" lib/      # expect nothing
grep -n "FlutterFragmentActivity" android/app/src/main/kotlin -r
```

## Gotchas

1. **`The current Activity must be a FragmentActivity`.** Symptom: `local_auth` throws `uiUnavailable` on Android, and no prompt appears. Cause: `MainActivity` extends `FlutterActivity`. Fix: extend `FlutterFragmentActivity`.
2. **A crash at launch on Android 8.** Symptom: an inflate error with the biometric plugin. Cause: `LaunchTheme` isn't an AppCompat theme. Fix: use a `Theme.AppCompat` parent.
3. **Wrong password shows a generic error.** Symptom: the `user-not-found` branch never runs. Cause: enumeration protection returns `invalid-credential`. Fix: map all three codes together.
4. **The login redirect never fires.** Symptom: state changes, but the screen stays. Cause: `GoRouter` has no `refreshListenable`. Fix: bump a `ValueNotifier` from `ref.listen`.
5. **`The name 'AuthState' is defined in the libraries`.** Symptom: an ambiguous import, with `User` or `OAuthProvider` too. Cause: `firebase_auth` and `supabase_flutter` share those names. Fix: keep each provider in its own file, or use `show`.
6. **Google sign-in misbehaves after a hot restart or a second screen.** Symptom: calls fail once two places set it up. Cause: `initialize` must run exactly once. Fix: cache the future, as the data source does.
7. **`serverClientId must be provided on Android`.** Symptom: a `clientConfigurationError` from `authenticate`. Cause: no web client ID. Fix: pass `serverClientId` to `initialize`, or configure it through `google-services.json`.
8. **App Review rejects under 4.8.** Symptom: rejected though Google login works. Cause: no equivalent privacy login. Fix: add a qualifying service, such as Sign in with Apple.

## Anti-Patterns

- Deciding entitlements or admin rights from a client flag. The server checks the token.
- Telling the user which part was wrong. "Email or password is incorrect" gives an attacker nothing.
- Verifying a TOTP code or backup code in the app. The secret would have to live on the device.
- Using `local_auth` alone to guard a payment. Bind the secret to the Keystore or Keychain.
- Waiting for `signOut` before clearing local tokens. A network error then leaves them on the device.

## Pre-Merge Checklist

- [ ] Every `AppAuthState` case is handled, with no default branch
- [ ] Tokens are in secure storage only, including the Supabase session
- [ ] `MainActivity` is a `FlutterFragmentActivity`, and `LaunchTheme` is AppCompat
- [ ] `NSFaceIDUsageDescription` is present and specific
- [ ] An equivalent privacy login exists where guideline 4.8 applies
- [ ] Account deletion works inside the app
- [ ] `invalid-credential`, cancellation and offline are each handled
- [ ] Logout clears tokens, cache and subscriptions even when offline
- [ ] The PIN uses salted Argon2id, and the attempt limit survives a restart and deletes the PIN at five failures
- [ ] MFA codes are verified on a server

## See Also

- `flutter-security` for pinning, Keystore-bound keys and release hardening
- `flutter-setup-declarative-routing` for the full route tree and deep links
- `flutter-state` for app lock providers
- `flutter-analytics-firebase` for consent before any event
