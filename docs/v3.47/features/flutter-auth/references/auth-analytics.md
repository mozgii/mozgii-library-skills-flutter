---
title: "Firebase Analytics events for Flutter auth"
author: MOZGII, Inc.
lang: en
tags: [flutter, auth, analytics, firebase]
description: A complete AuthNotifier that logs each auth step to Firebase Analytics with no personal data, and the event reference table.
---
# Firebase Analytics Auth Events

Log each auth transition from one place, the notifier, so screens stay free of tracking calls. Event names are snake_case, matching Firebase Analytics conventions, and no event carries an email, name or token. These calls go through consent: analytics stays off until the user agrees.

```dart
// lib/features/auth/presentation/tracked_auth_notifier.dart
import 'dart:async';

import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'tracked_auth_notifier.g.dart';

sealed class SessionState {
  const SessionState();
}

final class SessionLoading extends SessionState {
  const SessionLoading();
}

final class SessionSignedOut extends SessionState {
  const SessionSignedOut();
}

final class SessionSignedIn extends SessionState {
  const SessionSignedIn(this.userId);

  final String userId;
}

final class SessionFailed extends SessionState {
  const SessionFailed(this.code);

  final String code;
}

@Riverpod(keepAlive: true)
FirebaseAuth firebaseAuth(Ref ref) => FirebaseAuth.instance;

@Riverpod(keepAlive: true)
FirebaseAnalytics analytics(Ref ref) => FirebaseAnalytics.instance;

@Riverpod(keepAlive: true)
class TrackedAuthNotifier extends _$TrackedAuthNotifier {
  @override
  SessionState build() {
    final subscription = ref
        .watch(firebaseAuthProvider)
        .authStateChanges()
        .listen((user) {
      state = user == null
          ? const SessionSignedOut()
          : SessionSignedIn(user.uid);
    });
    ref.onDispose(subscription.cancel);
    return const SessionLoading();
  }

  FirebaseAnalytics get _analytics => ref.read(analyticsProvider);

  Future<void> signInWithEmail(String email, String password) async {
    state = const SessionLoading();
    try {
      await ref
          .read(firebaseAuthProvider)
          .signInWithEmailAndPassword(email: email, password: password);
      await _analytics.logLogin(loginMethod: 'email');
    } on FirebaseAuthException catch (e) {
      state = SessionFailed(e.code);
    }
  }

  Future<void> signUpWithEmail(String email, String password) async {
    state = const SessionLoading();
    try {
      await ref
          .read(firebaseAuthProvider)
          .createUserWithEmailAndPassword(email: email, password: password);
      await _analytics.logSignUp(signUpMethod: 'email');
    } on FirebaseAuthException catch (e) {
      state = SessionFailed(e.code);
    }
  }

  /// Call after any other provider succeeds, with `google` or `apple`.
  Future<void> recordLogin(String method) =>
      _analytics.logLogin(loginMethod: method);

  /// [result] is `success`, `failed` or `cancelled`.
  Future<void> recordBiometricPrompt(String result) => _analytics.logEvent(
        name: 'biometric_prompt',
        parameters: {'result': result},
      );

  /// [method] is `sms` or `totp`.
  Future<void> recordMfaTriggered(String method) => _analytics.logEvent(
        name: 'mfa_triggered',
        parameters: {'method': method},
      );

  Future<void> signOut() async {
    await ref.read(firebaseAuthProvider).signOut();
    await _analytics.logEvent(name: 'sign_out');
  }
}
```

## Auth Event Reference

| Event | Method | Parameters |
|---|---|---|
| Sign in success | `logLogin(loginMethod:)` | `login_method`: `email`, `google`, `apple`, `biometric`, `pin` |
| Sign up success | `logSignUp(signUpMethod:)` | `sign_up_method`: `email`, `google`, `apple` |
| Sign out | `logEvent(name: 'sign_out')` | none |
| Biometric prompt | `logEvent(name: 'biometric_prompt')` | `result`: `success`, `failed`, `cancelled` |
| MFA triggered | `logEvent(name: 'mfa_triggered')` | `method`: `sms`, `totp` |
