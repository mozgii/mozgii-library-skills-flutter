---
title: Flutter gradient cards, custom painters and dark mode rules
author: MOZGII, Inc.
lang: en
tags: [flutter, ui, custompainter, dark-mode]
description: A gradient card, a CustomPainter for non-standard shapes, and the rules that keep a screen correct in dark mode.
---
# Flutter Gradient Cards, Custom Painters and Dark Mode Rules

Custom surfaces and dark mode share one rule: draw from values the theme gave you, never from fixed colors.

## Gradient card from theme colors

A gradient built from `colorScheme` roles follows the theme and dynamic color with no extra work. Check text contrast against both ends of the gradient.

```dart
// lib/shared/widgets/gradient_card.dart
import 'package:material_ui/material_ui.dart';

import '../../core/theme/app_tokens.dart';

class GradientCard extends StatelessWidget {
  const GradientCard({super.key, required this.child});

  final Widget child;

  @override
  Widget build(BuildContext context) {
    final colors = Theme.of(context).colorScheme;
    return DecoratedBox(
      decoration: BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [colors.primary, colors.tertiary],
        ),
        borderRadius: BorderRadius.circular(AppRadius.xl),
        boxShadow: [
          BoxShadow(
            color: colors.primary.withValues(alpha: 0.3),
            blurRadius: 20,
            offset: const Offset(0, 8),
          ),
        ],
      ),
      child: Padding(padding: const EdgeInsets.all(AppSpacing.base), child: child),
    );
  }
}
```

## CustomPainter for non-standard shapes

Use `CustomPainter` for a shape `BoxDecoration` can't draw. Have `shouldRepaint` return `false` when inputs haven't changed, which prevents needless repaints.

```dart
import 'package:material_ui/material_ui.dart';

class WavePainter extends CustomPainter {
  WavePainter({required this.color, required this.progress});

  final Color color;
  final double progress; // 0 to 1

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = color
      ..style = PaintingStyle.fill;
    final level = size.height * (1 - progress);
    final path = Path()
      ..moveTo(0, level)
      ..quadraticBezierTo(
        size.width * 0.25,
        size.height * (0.8 - progress),
        size.width * 0.5,
        level,
      )
      ..quadraticBezierTo(
        size.width * 0.75,
        size.height * (1.2 - progress),
        size.width,
        level,
      )
      ..lineTo(size.width, size.height)
      ..lineTo(0, size.height)
      ..close();
    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(WavePainter oldDelegate) =>
      oldDelegate.progress != progress || oldDelegate.color != color;
}
```

## Semantic colors keep a screen correct in dark mode

Use color roles from the theme. `Colors.white` with `Colors.black87` looks fine until the theme flips, then fails silently. The pair below adapts.

```dart
import 'package:material_ui/material_ui.dart';

class ThemedLabel extends StatelessWidget {
  const ThemedLabel({super.key, required this.label});

  final String label;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return ColoredBox(
      color: theme.colorScheme.surface,
      child: Text(
        label,
        style: theme.textTheme.bodyLarge?.copyWith(
          color: theme.colorScheme.onSurface,
        ),
      ),
    );
  }
}
```

## ThemeExtension lerp during a theme change

Give every custom `ThemeExtension` token explicit light and dark values, and implement `lerp` for each field. `MaterialApp` wraps the app in `AnimatedTheme`, so a mode change runs `lerp` over `kThemeAnimationDuration`, 200 ms by default, or `themeAnimationDuration` if you set it. No other code is needed.

## System brightness for one-off decisions

`MediaQuery.platformBrightnessOf(context)` reads the system preference with no provider. Use it only for local decisions such as a status bar style, because a person may have overridden the mode in your app, and then `Theme.of(context).brightness` is the answer. A toggle, its persistence and preventing a flash at startup all belong in your state layer: a notifier holding `ThemeMode`, fed into `MaterialApp.themeMode`.

## Dark mode variants of icons and images

Pick an asset by theme brightness, or tint a monochrome asset so one file serves both modes.

```dart
import 'package:material_ui/material_ui.dart';

class Logo extends StatelessWidget {
  const Logo({super.key});

  @override
  Widget build(BuildContext context) {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    return Image.asset(isDark ? 'assets/logo_dark.png' : 'assets/logo_light.png');
  }
}

class MonoIcon extends StatelessWidget {
  const MonoIcon({super.key});

  @override
  Widget build(BuildContext context) => ColorFiltered(
        colorFilter: ColorFilter.mode(
          Theme.of(context).colorScheme.onSurface,
          BlendMode.srcIn,
        ),
        child: Image.asset('assets/icon_mono.png'),
      );
}
```
