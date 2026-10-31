---
title: Flutter design tokens for screens
author: MOZGII, Inc.
lang: en
tags: [flutter, ui, tokens, touch-target]
description: The AppColors extension, spacing and radius constants, and MinTouchTarget that the design skill's examples use.
---
# Flutter Design Tokens for Screens

Screens need a small shared vocabulary. These tokens are the same classes `flutter-theme` defines, so a project using both skills keeps one copy.

## AppColors ThemeExtension for roles ColorScheme lacks

`AppColors` holds success, warning and glass colors. Brand and surface colors stay in `ColorScheme`, which keeps every Material widget in step with custom ones. All `onX` pairs pass WCAG AA. A theme must carry this extension, or `context.appColors` fails its null check.

```dart
// lib/core/theme/app_colors.dart
import 'package:material_ui/material_ui.dart';

@immutable
class AppColors extends ThemeExtension<AppColors> {
  const AppColors({
    required this.success,
    required this.onSuccess,
    required this.warning,
    required this.onWarning,
    required this.glassFill,
    required this.glassBorder,
  });

  final Color success;
  final Color onSuccess;
  final Color warning;
  final Color onWarning;
  final Color glassFill;
  final Color glassBorder;

  static const light = AppColors(
    success: Color(0xFF1E7E34),
    onSuccess: Color(0xFFFFFFFF),
    warning: Color(0xFFB45309),
    onWarning: Color(0xFFFFFFFF),
    glassFill: Color(0xCCFFFFFF),
    glassBorder: Color(0x33FFFFFF),
  );

  static const dark = AppColors(
    success: Color(0xFF4CAF76),
    onSuccess: Color(0xFF002A14),
    warning: Color(0xFFFFB347),
    onWarning: Color(0xFF3A1A00),
    glassFill: Color(0x991C1C2E),
    glassBorder: Color(0x1AFFFFFF),
  );

  static const highContrast = AppColors(
    success: Color(0xFF006600),
    onSuccess: Color(0xFFFFFFFF),
    warning: Color(0xFF8A3D00),
    onWarning: Color(0xFFFFFFFF),
    glassFill: Color(0xFFFFFFFF),
    glassBorder: Color(0xFF000000),
  );

  @override
  AppColors copyWith({
    Color? success,
    Color? onSuccess,
    Color? warning,
    Color? onWarning,
    Color? glassFill,
    Color? glassBorder,
  }) =>
      AppColors(
        success: success ?? this.success,
        onSuccess: onSuccess ?? this.onSuccess,
        warning: warning ?? this.warning,
        onWarning: onWarning ?? this.onWarning,
        glassFill: glassFill ?? this.glassFill,
        glassBorder: glassBorder ?? this.glassBorder,
      );

  @override
  AppColors lerp(AppColors? other, double t) {
    if (other == null) return this;
    return AppColors(
      success: Color.lerp(success, other.success, t)!,
      onSuccess: Color.lerp(onSuccess, other.onSuccess, t)!,
      warning: Color.lerp(warning, other.warning, t)!,
      onWarning: Color.lerp(onWarning, other.onWarning, t)!,
      glassFill: Color.lerp(glassFill, other.glassFill, t)!,
      glassBorder: Color.lerp(glassBorder, other.glassBorder, t)!,
    );
  }
}

extension AppColorsX on BuildContext {
  AppColors get appColors => Theme.of(this).extension<AppColors>()!;
}
```

## AppSpacing and AppRadius constants

Spacing and radius don't change with brightness, so plain constants beat an extension.

```dart
// lib/core/theme/app_tokens.dart
abstract final class AppSpacing {
  static const double xs = 4;
  static const double sm = 8;
  static const double md = 12;
  static const double base = 16;
  static const double lg = 24;
  static const double xl = 32;
  static const double xxl = 48;
  static const double xxxl = 64;
}

abstract final class AppRadius {
  static const double sm = 8;
  static const double md = 12;
  static const double lg = 16;
  static const double xl = 24;
  static const double full = 999;
}
```

## MinTouchTarget for custom tappables

`MinTouchTarget` grows a widget's hit area to 48 × 48 without changing what's drawn. Larger children keep their size, and a fixed `SizedBox` would clip them. 48 dp is Material's minimum and covers Apple's 44 pt too; at 3× it's 144 physical pixels, enough for reliable contact with a finger, including for people with motor impairments.

```dart
import 'package:material_ui/material_ui.dart';

class MinTouchTarget extends StatelessWidget {
  const MinTouchTarget({
    super.key,
    required this.child,
    this.onTap,
    this.semanticLabel,
    this.minSize = 48,
  });

  final Widget child;
  final VoidCallback? onTap;
  final String? semanticLabel;
  final double minSize;

  @override
  Widget build(BuildContext context) => Semantics(
        button: true,
        enabled: onTap != null,
        label: semanticLabel,
        child: GestureDetector(
          onTap: onTap,
          behavior: HitTestBehavior.opaque,
          child: ConstrainedBox(
            constraints: BoxConstraints(minWidth: minSize, minHeight: minSize),
            child: Center(widthFactor: 1, heightFactor: 1, child: child),
          ),
        ),
      );
}
```

## LiquidGlassCard: A Frosted Surface

`LiquidGlassCard` blurs what's behind it with `BackdropFilter`, fills with the theme's translucent glass colour, draws a thin light border and clips, so the blur stays inside the shape.

```dart
// lib/shared/widgets/liquid_glass_card.dart
import 'dart:ui' as ui;

import 'package:material_ui/material_ui.dart';

import '../../core/theme/app_colors.dart';
import '../../core/theme/app_tokens.dart';

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
```
