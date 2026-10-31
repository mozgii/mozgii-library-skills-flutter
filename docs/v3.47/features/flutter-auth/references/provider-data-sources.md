---
title: "Flutter auth data sources: biometric, Firebase, Google, Apple, Supabase"
author: MOZGII, Inc.
lang: en
tags: [flutter, auth, firebase, supabase, google-sign-in, sign-in-with-apple, local-auth]
description: Complete data source classes for each sign-in provider, analyzed against the pinned packages, with error mapping, the Apple nonce and a secure Supabase session store.
---
# Auth Data Sources by Provider

Each provider stays in its own file. `firebase_auth` and `supabase_flutter` both export `User` and `OAuthProvider`, and Supabase exports `AuthState`, so one file importing both gets ambiguous names.

## Biometric Data Source with local_auth

The source handles the two states a user can be in before a prompt, no hardware and nothing enrolled, then maps each failure code for the UI.

```dart
// lib/features/auth/data/biometric_data_source.dart
import 'package:local_auth/local_auth.dart';

enum BiometricCapability { available, notEnrolled, unavailable }

final class BiometricFailure implements Exception {
  const BiometricFailure(this.code, this.message);

  final LocalAuthExceptionCode code;
  final String message;
}

class BiometricDataSource {
  BiometricDataSource([LocalAuthentication? auth])
      : _auth = auth ?? LocalAuthentication();

  final LocalAuthentication _auth;

  Future<BiometricCapability> checkCapability() async {
    if (!await _auth.canCheckBiometrics) return BiometricCapability.unavailable;
    final enrolled = await _auth.getAvailableBiometrics();
    return enrolled.isEmpty
        ? BiometricCapability.notEnrolled
        : BiometricCapability.available;
  }

  Future<bool> authenticate({required String reason}) async {
    try {
      return await _auth.authenticate(
        localizedReason: reason,
        persistAcrossBackgrounding: true, // Keeps the prompt across app switches.
        biometricOnly: true, // No device PIN fallback; adjust per requirements.
      );
    } on LocalAuthException catch (e) {
      // Map noBiometricHardware, temporaryLockout and biometricLockout to your UI.
      throw BiometricFailure(e.code, e.description ?? e.code.name);
    }
  }
}
```

## Firebase Auth Data Source and Error Mapping

Email enumeration protection answers `invalid-credential` for a wrong password and an unknown address. The mapper folds the two older codes into the same result, so older projects behave alike.

```dart
// lib/features/auth/data/firebase_auth_data_source.dart
import 'package:firebase_auth/firebase_auth.dart';

enum AuthErrorCode {
  invalidCredentials,
  emailInUse,
  rateLimited,
  network,
  cancelled,
  unknown,
}

final class AppAuthException implements Exception {
  const AppAuthException(this.code, [this.detail]);

  final AuthErrorCode code;
  final String? detail;
}

AppAuthException mapFirebaseError(FirebaseAuthException e) => switch (e.code) {
      'invalid-credential' ||
      'user-not-found' ||
      'wrong-password' =>
        const AppAuthException(AuthErrorCode.invalidCredentials),
      'email-already-in-use' => const AppAuthException(AuthErrorCode.emailInUse),
      'too-many-requests' => const AppAuthException(AuthErrorCode.rateLimited),
      'network-request-failed' => const AppAuthException(AuthErrorCode.network),
      _ => AppAuthException(AuthErrorCode.unknown, e.code),
    };

class FirebaseAuthDataSource {
  FirebaseAuthDataSource(this._auth);

  final FirebaseAuth _auth;

  Stream<User?> get authStateChanges => _auth.authStateChanges();

  Future<void> signIn(String email, String password) async {
    try {
      await _auth.signInWithEmailAndPassword(email: email, password: password);
    } on FirebaseAuthException catch (e) {
      throw mapFirebaseError(e);
    }
  }

  Future<void> signUp(String email, String password) async {
    try {
      final result = await _auth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      await result.user?.sendEmailVerification();
    } on FirebaseAuthException catch (e) {
      throw mapFirebaseError(e);
    }
  }

  Future<void> signOut() => _auth.signOut();
}
```

## Google Sign-In Data Source

The class caches the `initialize` future, because the package says it must run exactly once, before any other call.

```dart
// lib/features/auth/data/google_auth_data_source.dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:yourapp/features/auth/data/firebase_auth_data_source.dart';

class GoogleAuthDataSource {
  GoogleAuthDataSource(this._auth, {this.serverClientId});

  final FirebaseAuth _auth;

  /// The web client ID from the Firebase console; Android needs it.
  final String? serverClientId;

  final GoogleSignIn _google = GoogleSignIn.instance;

  // initialize() must run exactly once, before any other call.
  static Future<void>? _initialized;
  Future<void> _ensureInitialized() =>
      _initialized ??= _google.initialize(serverClientId: serverClientId);

  Future<void> signIn() async {
    await _ensureInitialized();
    try {
      final account = await _google.authenticate(
        scopeHint: const ['email', 'profile'],
      );
      // `authentication` is synchronous in 7.x and carries only the ID token.
      final credential = GoogleAuthProvider.credential(
        idToken: account.authentication.idToken,
      );
      await _auth.signInWithCredential(credential);
    } on GoogleSignInException catch (e) {
      if (e.code == GoogleSignInExceptionCode.canceled) {
        throw const AppAuthException(AuthErrorCode.cancelled);
      }
      rethrow;
    }
  }

  Future<void> signOut() async {
    await _ensureInitialized();
    await _google.signOut();
  }
}
```

## Apple Sign In Data Source with a Nonce

The nonce ties the identity token to this attempt, so a captured token can't be replayed. Apple receives the hash and Firebase checks it against the raw value.

```dart
// lib/features/auth/data/apple_auth_data_source.dart
import 'dart:convert';
import 'dart:math';

import 'package:crypto/crypto.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

class AppleAuthDataSource {
  AppleAuthDataSource(this._auth);

  final FirebaseAuth _auth;

  Future<void> signIn() async {
    // Apple receives the hash; Firebase checks it against the raw value.
    final rawNonce = _randomNonce();
    final hashedNonce = sha256.convert(utf8.encode(rawNonce)).toString();

    final apple = await SignInWithApple.getAppleIDCredential(
      scopes: [
        AppleIDAuthorizationScopes.email,
        AppleIDAuthorizationScopes.fullName,
      ],
      nonce: hashedNonce,
    );
    final idToken = apple.identityToken;
    if (idToken == null) throw StateError('Apple returned no identity token');

    final credential = AppleAuthProvider.credentialWithIDToken(
      idToken,
      rawNonce,
      AppleFullPersonName(
        givenName: apple.givenName,
        familyName: apple.familyName,
      ),
    );
    await _auth.signInWithCredential(credential);
  }

  String _randomNonce() {
    final random = Random.secure();
    return base64Url.encode(List<int>.generate(32, (_) => random.nextInt(256)));
  }
}
```

## Supabase Data Source and Secure Session Store

The default store is `SharedPreferences`. `SecureSessionStorage` moves the session to the Keychain and Keystore, and the `supabasePersistSessionKey` constant comes from the package.

```dart
// lib/features/auth/data/supabase_setup.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

class SecureSessionStorage extends LocalStorage {
  const SecureSessionStorage();

  static const _storage = FlutterSecureStorage();

  @override
  Future<void> initialize() async {}

  @override
  Future<bool> hasAccessToken() =>
      _storage.containsKey(key: supabasePersistSessionKey);

  @override
  Future<String?> accessToken() => _storage.read(key: supabasePersistSessionKey);

  @override
  Future<void> persistSession(String persistSessionString) => _storage.write(
        key: supabasePersistSessionKey,
        value: persistSessionString,
      );

  @override
  Future<void> removePersistedSession() =>
      _storage.delete(key: supabasePersistSessionKey);
}

Future<void> initSupabase(String url, String publishableKey) =>
    Supabase.initialize(
      url: url,
      publishableKey: publishableKey,
      authOptions: const FlutterAuthClientOptions(
        localStorage: SecureSessionStorage(),
      ),
    );

class SupabaseAuthDataSource {
  SupabaseAuthDataSource(this._client);

  final SupabaseClient _client;

  Stream<AuthState> get authStateChanges => _client.auth.onAuthStateChange;

  Future<AuthResponse> signIn(String email, String password) =>
      _client.auth.signInWithPassword(email: email, password: password);

  Future<bool> signInWithGoogle() => _client.auth.signInWithOAuth(
        OAuthProvider.google,
        redirectTo: 'com.example.app://login-callback',
      );

  Future<void> signOut() => _client.auth.signOut();

  Session? get currentSession => _client.auth.currentSession;
}
```

## Android Files for the local_auth Prompt

The `local_auth` prompt on Android needs `MainActivity` to extend `FlutterFragmentActivity` and `LaunchTheme` to have a `Theme.AppCompat` parent.

```kotlin
// android/app/src/main/kotlin/com/example/app/MainActivity.kt
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity : FlutterFragmentActivity()
```

```xml
<!-- android/app/src/main/res/values/styles.xml -->
<style name="LaunchTheme" parent="Theme.AppCompat.DayNight.NoActionBar">
    <item name="android:windowBackground">@drawable/launch_background</item>
</style>
```
