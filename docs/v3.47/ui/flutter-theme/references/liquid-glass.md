---
title: Liquid Glass look in Flutter
author: MOZGII, Inc.
lang: en
tags: [flutter, theme, liquid-glass, ios]
description: A BackdropFilter glass card, the cost of blur, the real native option on iOS 26, and how to read the iOS version safely.
---
# Liquid Glass Look in Flutter

Apple's Liquid Glass belongs to UIKit: `UIGlassEffect` arrived in iOS 26 and renders through `UIVisualEffectView`. Flutter paints its own surface, so a Flutter widget can't sample that effect. You have two honest options.

| Option | Fits when | Costs |
|---|---|---|
| `BackdropFilter` card | Almost always: cards, sheets, toolbars on every platform | An approximation, with no real vibrancy or refraction |
| Native `UIVisualEffectView` in a `UiKitView` | A single hero surface on iOS 26+ where the real material matters | A platform view per surface, Swift code, and a fallback for everything else |

This library doesn't ship a native bridge, because a stub that falls back silently is worse than none. If you build one, keep the widget's API and swap only the implementation on iOS 26 and later.

## LiquidGlassCard with BackdropFilter

This card uses `BackdropFilter` on every platform, reads its colors from `AppColors`, and clips so the blur stays inside the shape. With `AppColors.highContrast` the fill is opaque, so the effect disappears for people who asked for contrast.

```dart
// lib/core/theme/liquid_glass.dart
import 'dart:ui' as ui;

import 'package:material_ui/material_ui.dart';

import 'app_colors.dart';
import 'app_tokens.dart';
import 'min_touch_target.dart';

class LiquidGlassCard extends StatelessWidget {
  const LiquidGlassCard({
    super.key,
    required this.child,
    this.borderRadius = const BorderRadius.all(Radius.circular(AppRadius.xl)),
    this.padding = const EdgeInsets.all(AppSpacing.base),
    this.blur = 24,
  });

  final Widget child;
  final BorderRadius borderRadius;
  final EdgeInsetsGeometry padding;
  final double blur;

  @override
  Widget build(BuildContext context) {
    final colors = context.appColors;
    return ClipRRect(
      borderRadius: borderRadius,
      child: BackdropFilter(
        filter: ui.ImageFilter.blur(sigmaX: blur, sigmaY: blur),
        child: DecoratedBox(
          decoration: BoxDecoration(
            color: colors.glassFill,
            borderRadius: borderRadius,
            border: Border.all(color: colors.glassBorder),
          ),
          child: Padding(padding: padding, child: child),
        ),
      ),
    );
  }
}

class LiquidGlassButton extends StatelessWidget {
  const LiquidGlassButton({
    super.key,
    required this.onPressed,
    required this.child,
    this.blur = 20,
  });

  final VoidCallback? onPressed;
  final Widget child;
  final double blur;

  @override
  Widget build(BuildContext context) => MinTouchTarget(
        onTap: onPressed,
        child: LiquidGlassCard(
          blur: blur,
          padding: const EdgeInsets.symmetric(
            horizontal: AppSpacing.lg,
            vertical: AppSpacing.md,
          ),
          child: child,
        ),
      );
}
```

## Cost of BackdropFilter blur

Every `BackdropFilter` reads back what's behind it and blurs it on the raster thread. Cost grows with its area and with how many are on screen, and Flutter's renderer does this work on iOS too: Core Animation isn't involved. Keep to two or three per frame on low-end Android. For several glass cards in one screen, wrap them in a `BackdropGroup` and use `BackdropFilter.grouped` so they share one backdrop input, which is cheaper. Check in DevTools that the raster phase stays inside budget.

## Reading the iOS version for a native tier

`Platform.operatingSystemVersion` is the wrong tool. Dart documents its format as unsuited to parsing, on Apple platforms it comes from a human-readable string shaped like `Version 14.5 (Build 18E182)` rather than `26.0`, and on the simulator it has been reported to give the host's version. `device_info_plus` returns the real one. It's a platform call, so read it once at startup.

```dart
import 'package:device_info_plus/device_info_plus.dart';
import 'package:flutter/foundation.dart';

Future<bool> supportsNativeGlass() async {
  if (kIsWeb || defaultTargetPlatform != TargetPlatform.iOS) return false;
  final ios = await DeviceInfoPlugin().iosInfo;
  final major = int.tryParse(ios.systemVersion.split('.').first);
  return major != null && major >= 26;
}
```
