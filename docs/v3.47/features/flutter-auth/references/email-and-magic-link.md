---
title: "Flutter email verification and Supabase magic links"
author: MOZGII, Inc.
lang: en
tags: [flutter, auth, email, supabase, deep-link]
description: The email verification screen for Firebase, and passwordless magic links with Supabase including the deep link setup, which supabase_flutter handles itself once the platforms are configured.
---
# Email Verification and Magic Links

## Firebase Email Verification Flow

Firebase sets `emailVerified` when the user opens the link, but the app learns of it only by reloading the user: `authStateChanges` stays silent for that change. The screen below offers a check button and a resend with a 60-second cooldown. The auth state has an `AuthRequiresEmailVerification` case, and the router sends that state to `/auth/verify-email`.

```dart
// lib/features/auth/presentation/email_verification_page.dart
import 'dart:async';

import 'package:firebase_auth/firebase_auth.dart';
import 'package:material_ui/material_ui.dart';

class EmailVerificationPage extends StatefulWidget {
  const EmailVerificationPage({
    super.key,
    required this.user,
    required this.onVerified,
  });

  final User user;

  /// Hand the reloaded user to your auth notifier here.
  final ValueChanged<User> onVerified;

  @override
  State<EmailVerificationPage> createState() => _EmailVerificationPageState();
}

class _EmailVerificationPageState extends State<EmailVerificationPage> {
  static const _cooldownSeconds = 60;

  Timer? _timer;
  int _secondsLeft = 0;
  String? _message;

  void _startCooldown() {
    setState(() => _secondsLeft = _cooldownSeconds);
    _timer?.cancel();
    _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
      setState(() => _secondsLeft--);
      if (_secondsLeft == 0) timer.cancel();
    });
  }

  Future<void> _resend() async {
    await widget.user.sendEmailVerification();
    if (mounted) _startCooldown();
  }

  Future<void> _check() async {
    await widget.user.reload();
    final fresh = FirebaseAuth.instance.currentUser;
    if (!mounted) return;
    if (fresh?.emailVerified ?? false) {
      widget.onVerified(fresh!);
    } else {
      setState(() => _message = 'Not verified yet. Check your inbox.');
    }
  }

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final text = Theme.of(context).textTheme;
    return Scaffold(
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.mark_email_unread_outlined, size: 72),
              const SizedBox(height: 16),
              Text('Verify your email', style: text.headlineMedium),
              const SizedBox(height: 8),
              Text(_message ?? 'Check your inbox and click the link.'),
              const SizedBox(height: 24),
              FilledButton(
                onPressed: _check,
                child: const Text('I have verified'),
              ),
              const SizedBox(height: 8),
              TextButton(
                onPressed: _secondsLeft > 0 ? null : _resend,
                child: Text(
                  _secondsLeft > 0
                      ? 'Resend in ${_secondsLeft}s'
                      : 'Resend verification email',
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## Supabase Magic Link

A magic link signs in with an email and no password. `supabase_flutter` uses `app_links` internally: once the platform accepts the link, its observer runs `getSessionFromUrl` and emits `signedIn` on `onAuthStateChange`. Don't call `getSessionFromUrl` yourself or add a second `app_links` listener for this: the code in a PKCE link works once, so a duplicate exchange fails. `detectSessionInUri` defaults to true.

```dart
// lib/features/auth/data/magic_link_data_source.dart
import 'package:supabase_flutter/supabase_flutter.dart';

class MagicLinkDataSource {
  MagicLinkDataSource(this._client);

  final SupabaseClient _client;

  Future<void> send(String email) => _client.auth.signInWithOtp(
        email: email,
        emailRedirectTo: 'com.example.app://login-callback',
      );

  /// The session arrives here after the link opens the app.
  Stream<Session> get signedInSessions => _client.auth.onAuthStateChange
      .where((state) => state.event == AuthChangeEvent.signedIn)
      .map((state) => state.session)
      .where((session) => session != null)
      .cast<Session>();
}
```

## Deep Link Setup for Supabase Redirects

Add `com.example.app://login-callback` to the redirect URLs in the Supabase dashboard. Use a reverse-domain scheme, because a custom scheme must be unique on the device. Each platform then has to route the link to the app.

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>com.example.app</string>
    </array>
  </dict>
</array>
```

```xml
<!-- android/app/src/main/AndroidManifest.xml, inside the main activity -->
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="com.example.app" android:host="login-callback" />
</intent-filter>
```

`android:autoVerify` applies to `https` App Links, not a custom scheme, so leave it out here. Any app can register a custom scheme, which PKCE guards against: the stolen code is useless without the verifier held by your app. For the strongest setup, use a verified `https` link instead.

If your app also runs `app_links` for its own routes, restrict Supabase with `detectSessionInUriPredicate`, so only auth callbacks are consumed.

## Gotchas for Links and Email

1. **The link opens the app, and nothing happens.** Symptom: no session after the tap. Cause: the redirect URL isn't allowed in the dashboard. Fix: add the exact URL.
2. **`Code verifier could not be found`.** Symptom: an error after opening a link. Cause: the link was opened on another device, or the app data was cleared. Fix: send the user back to request a new link.
3. **Two exchanges for one link.** Symptom: a session appears, then an error. Cause: a second listener called `getSessionFromUrl`. Fix: remove it.
