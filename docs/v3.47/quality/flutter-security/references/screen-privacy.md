---
title: "Flutter app-switcher privacy cover without a plugin"
author: MOZGII, Inc.
lang: en
tags: [flutter, security, privacy, ios, lifecycle]
description: A Dart-only cover that hides app content while the app is inactive, and when to prefer the native screen_protector calls instead.
---
# App-Switcher Privacy Cover in Dart

An app-switcher cover hides the screen while the app is inactive, so the OS snapshot doesn't capture it. `screen_protector`'s `protectDataLeakageWithBlur` and `protectDataLeakageWithColor` do this natively on iOS, and that is the better choice: the OS can snapshot before Flutter has drawn another frame, so a Dart cover can arrive too late. Use this version when the app can't take the plugin, and test the switcher on a real device.

Place the cover in `MaterialApp.builder` so it sits above every route. It uses a `Stack` rather than an `OverlayEntry`, because `builder` runs above the `Navigator`, where no `Overlay` exists yet.

```dart
// lib/core/security/privacy_cover.dart
import 'package:material_ui/material_ui.dart';

class PrivacyCover extends StatefulWidget {
  const PrivacyCover({super.key, required this.child});

  final Widget child;

  @override
  State<PrivacyCover> createState() => _PrivacyCoverState();
}

class _PrivacyCoverState extends State<PrivacyCover> {
  bool _covered = false;
  late final AppLifecycleListener _lifecycle;

  @override
  void initState() {
    super.initState();
    _lifecycle = AppLifecycleListener(
      onInactive: () => setState(() => _covered = true),
      onResume: () => setState(() => _covered = false),
    );
  }

  @override
  void dispose() {
    _lifecycle.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      fit: StackFit.expand,
      children: [
        widget.child,
        if (_covered)
          ColoredBox(
            color: Theme.of(context).colorScheme.surface,
            child: const Center(child: Icon(Icons.lock, size: 64)),
          ),
      ],
    );
  }
}
```

```dart
// lib/app.dart
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/core/security/privacy_cover.dart';

class App extends StatelessWidget {
  const App({super.key, required this.home});

  final Widget home;

  @override
  Widget build(BuildContext context) => MaterialApp(
        home: home,
        builder: (context, child) =>
            PrivacyCover(child: child ?? const SizedBox.shrink()),
      );
}
```

On iOS, `onInactive` also fires for a Control Center swipe or a system dialog such as Face ID, so the cover flashes during a biometric prompt. If that matters, track an `isAuthenticating` flag and skip the cover while it's set.
