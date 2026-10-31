---
title: "Flutter MFA: SMS codes, TOTP and backup codes"
author: MOZGII, Inc.
lang: en
tags: [flutter, auth, mfa, totp, otp, firebase]
description: Second factors for a Flutter app - Firebase Phone Auth, TOTP through Firebase or Supabase, and a custom backend flow, with every secret and verification held on the server.
---
# MFA and One-Time Codes in Flutter

A second factor protects the server, so the server has to do the checking. An app that verifies a TOTP code must hold the shared secret, and anyone who reads that secret off the device can produce codes forever. Patching one `if` skips the check entirely. The same goes for backup codes: hashes in `flutter_secure_storage` guard nothing, because the attacker's device decides what passes.

| Backend | TOTP | Who verifies |
|---|---|---|
| Firebase with Identity Platform | `TotpMultiFactorGenerator` | Firebase |
| Supabase | `auth.mfa.enroll`, `auth.mfa.challengeAndVerify` | Supabase |
| Your own API | Enrollment and verify endpoints | Your server, for example with the Dart `otp` package |

## TOTP with Firebase Multi-Factor

Firebase generates the secret and checks the code. The app only shows the QR code and sends what the user types. TOTP requires the Identity Platform upgrade on the project.

```dart
// lib/features/auth/data/firebase_totp.dart
import 'package:firebase_auth/firebase_auth.dart';

class FirebaseTotp {
  FirebaseTotp(this._user);

  final User _user;

  /// Step 1: returns the secret to display. It stays in memory, never in storage.
  Future<TotpSecret> start() async {
    final session = await _user.multiFactor.getSession();
    return TotpMultiFactorGenerator.generateSecret(session);
  }

  /// Step 2: the otpauth:// value for the QR code.
  Future<String> qrCodeUri(TotpSecret secret, {required String issuer}) =>
      secret.generateQrCodeUrl(accountName: _user.email, issuer: issuer);

  /// Step 3: the server checks the first code, then enables the factor.
  Future<void> confirm(TotpSecret secret, String code) async {
    final assertion = await TotpMultiFactorGenerator.getAssertionForEnrollment(
      secret,
      code,
    );
    await _user.multiFactor.enroll(assertion, displayName: 'Authenticator app');
  }
}
```

## TOTP with Supabase

```dart
// lib/features/auth/data/supabase_totp.dart
import 'package:supabase_flutter/supabase_flutter.dart';

class SupabaseTotp {
  SupabaseTotp(this._client);

  final SupabaseClient _client;

  Future<AuthMFAEnrollResponse> enroll() =>
      _client.auth.mfa.enroll(factorType: FactorType.totp);

  Future<void> verify(String factorId, String code) async {
    await _client.auth.mfa.challengeAndVerify(factorId: factorId, code: code);
  }
}
```

## TOTP with Your Own Server

The server creates a 20-byte random secret, stores it encrypted, and returns the `otpauth://` URI once. The URI format is `otpauth://totp/<account>?secret=<base32>&issuer=<app>&algorithm=SHA1&digits=6&period=30`. Verification on the server allows one 30-second period of clock drift each way, refuses a code it has already accepted, and limits attempts per account. Don't mark the factor active until the first code passes.

```dart
// lib/features/auth/data/totp_api.dart
import 'package:dio/dio.dart';

final class TotpEnrollment {
  const TotpEnrollment({required this.otpauthUri, required this.manualKey});

  final String otpauthUri;
  final String manualKey;
}

class TotpApi {
  TotpApi(this._dio);

  final Dio _dio;

  Future<TotpEnrollment> startEnrollment() async {
    final response = await _dio.post<Map<String, dynamic>>('/mfa/totp/setup');
    final data = response.data!;
    return TotpEnrollment(
      otpauthUri: data['otpauth_uri'] as String,
      manualKey: data['manual_key'] as String,
    );
  }

  /// Returns backup codes, shown once and never stored in the app.
  Future<List<String>> confirmEnrollment(String code) async {
    final response = await _dio.post<Map<String, dynamic>>(
      '/mfa/totp/confirm',
      data: {'code': code},
    );
    return (response.data!['backup_codes'] as List<dynamic>).cast<String>();
  }
}
```

### TOTP Setup Screen

```dart
// lib/features/auth/presentation/totp_setup_page.dart
import 'package:material_ui/material_ui.dart';
import 'package:pinput/pinput.dart';
import 'package:qr_flutter/qr_flutter.dart';
import 'package:yourapp/features/auth/data/totp_api.dart';

class TotpSetupPage extends StatefulWidget {
  const TotpSetupPage({super.key, required this.api, required this.onEnrolled});

  final TotpApi api;
  final ValueChanged<List<String>> onEnrolled;

  @override
  State<TotpSetupPage> createState() => _TotpSetupPageState();
}

class _TotpSetupPageState extends State<TotpSetupPage> {
  late final Future<TotpEnrollment> _enrollment = widget.api.startEnrollment();
  String? _error;

  Future<void> _confirm(String code) async {
    try {
      widget.onEnrolled(await widget.api.confirmEnrollment(code));
    } on Exception {
      if (mounted) setState(() => _error = 'Invalid code. Try again.');
    }
  }

  @override
  Widget build(BuildContext context) {
    final text = Theme.of(context).textTheme;
    return Scaffold(
      appBar: AppBar(title: const Text('Set up authenticator')),
      body: FutureBuilder<TotpEnrollment>(
        future: _enrollment,
        builder: (context, snapshot) {
          if (snapshot.hasError) {
            return const Center(child: Text('Could not start setup.'));
          }
          final enrollment = snapshot.data;
          if (enrollment == null) {
            return const Center(child: CircularProgressIndicator());
          }
          return ListView(
            padding: const EdgeInsets.all(24),
            children: [
              Text(
                'Scan this code with your authenticator app.',
                style: text.bodyLarge,
              ),
              const SizedBox(height: 24),
              Center(
                child: QrImageView(
                  data: enrollment.otpauthUri,
                  size: 200,
                  backgroundColor: Colors.white,
                  semanticsLabel: 'Authenticator setup code',
                ),
              ),
              const SizedBox(height: 16),
              SelectableText(
                'Manual entry: ${enrollment.manualKey}',
                style: text.bodySmall,
              ),
              const SizedBox(height: 32),
              Pinput(
                length: 6,
                errorText: _error,
                forceErrorState: _error != null, // or errorText stays hidden
                onCompleted: _confirm,
              ),
            ],
          );
        },
      ),
    );
  }
}
```

## Backup Codes on the Server

Backup codes follow the same rule. The server generates eight codes, each 5 random bytes shown as 10 hex characters, stores only slow hashes of them, and returns the plain codes once. Each code works one time. The app displays them, offers a copy, and keeps nothing.

## Firebase Phone Auth with SMS Codes

`verifyPhoneNumber` is callback-driven. On Android, Firebase can read the SMS through the SMS Retriever API, with no `READ_SMS` permission, and it calls `verificationCompleted` with no user input. Numbers must be in E.164 format such as `+12025551234`, so validate that first. Show a 60-second countdown after `codeSent` with `Timer.periodic`, and keep the resend button off during it. This flow signs in with a phone number; using a phone as a second factor goes through `PhoneMultiFactorGenerator` and `user.multiFactor.enroll`.

```dart
// lib/features/auth/data/phone_auth_data_source.dart
import 'package:firebase_auth/firebase_auth.dart';

class PhoneAuthDataSource {
  PhoneAuthDataSource(this._auth);

  final FirebaseAuth _auth;

  String? _verificationId;
  int? _resendToken;

  Future<void> start({
    required String phoneNumber,
    required void Function() onCodeSent,
    required void Function() onAutoVerified,
    required void Function(FirebaseAuthException error) onError,
  }) {
    return _auth.verifyPhoneNumber(
      phoneNumber: phoneNumber,
      forceResendingToken: _resendToken,
      timeout: const Duration(seconds: 60),
      verificationCompleted: (credential) async {
        await _auth.signInWithCredential(credential);
        onAutoVerified();
      },
      verificationFailed: onError,
      codeSent: (verificationId, resendToken) {
        _verificationId = verificationId;
        _resendToken = resendToken;
        onCodeSent();
      },
      codeAutoRetrievalTimeout: (verificationId) {
        // Auto-retrieval ended; the id still works for manual entry.
        _verificationId = verificationId;
      },
    );
  }

  Future<UserCredential> confirm(String smsCode) {
    final verificationId = _verificationId;
    if (verificationId == null) throw StateError('No code was requested');
    return _auth.signInWithCredential(
      PhoneAuthProvider.credential(
        verificationId: verificationId,
        smsCode: smsCode,
      ),
    );
  }
}
```

Resending is another call to `start`: the stored `forceResendingToken` makes Firebase treat it as a resend.

## Checks for Any Second Factor

- [ ] No secret, hash or code comparison exists in the app
- [ ] The server limits attempts per account and refuses a reused TOTP code
- [ ] The factor is active only after the first code passes
- [ ] Backup codes appear once and work once
- [ ] Losing the device has a documented recovery path
