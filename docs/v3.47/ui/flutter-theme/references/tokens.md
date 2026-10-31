---
title: Flutter theme token reference
author: MOZGII, Inc.
lang: en
tags: [flutter, theme, tokens, color-scheme]
description: Spacing, elevation and radius constants, the full brand ColorScheme palette for light, dark and high contrast, and an AppTypography extension with complete copyWith and lerp.
---
# Flutter Theme Token Reference

Tokens carry names for decisions. This reference holds the two larger sets: a brand palette expressed as a `ColorScheme`, and a typography extension for text roles `TextTheme` lacks.

## AppSpacing, AppElevation and AppRadius constants

These values don't change with brightness, so plain constants beat an extension: nothing needs `lerp`, and `const` works everywhere.

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

abstract final class AppElevation {
  static const double none = 0;
  static const double xs = 1;
  static const double sm = 2;
  static const double md = 4;
  static const double lg = 8;
  static const double xl = 16;
}

abstract final class AppRadius {
  static const double sm = 8;
  static const double md = 12;
  static const double lg = 16;
  static const double xl = 24;
  static const double full = 999;
}
```

## Brand ColorScheme palette

Pass these to `FlexThemeData.light(colorScheme: ...)` instead of a built-in `FlexScheme` when a brand has fixed colors. All `onX` pairs below pass WCAG AA.

```dart
// lib/core/theme/brand_schemes.dart
import 'package:material_ui/material_ui.dart';

const brandLight = ColorScheme(
  brightness: Brightness.light,
  primary: Color(0xFF1B6EF3),
  onPrimary: Color(0xFFFFFFFF),
  secondary: Color(0xFF6C47FF),
  onSecondary: Color(0xFFFFFFFF),
  error: Color(0xFFD93025),
  onError: Color(0xFFFFFFFF),
  surface: Color(0xFFF8F9FC),
  onSurface: Color(0xFF1A1A2E),
  surfaceContainerHighest: Color(0xFFF4F5F7),
  outline: Color(0xFFD0D5DD),
  outlineVariant: Color(0xFFE8ECF0),
  scrim: Color(0x66000000),
  inverseSurface: Color(0xFF1A1A2E),
  onInverseSurface: Color(0xFFFFFFFF),
);

const brandDark = ColorScheme(
  brightness: Brightness.dark,
  primary: Color(0xFF5B9BFF),
  onPrimary: Color(0xFF001A40),
  secondary: Color(0xFF9E7EFF),
  onSecondary: Color(0xFF1A0066),
  error: Color(0xFFFF6B6B),
  onError: Color(0xFF3A0000),
  surface: Color(0xFF0F0F1A),
  onSurface: Color(0xFFE8E8F0),
  surfaceContainerHighest: Color(0xFF252538),
  outline: Color(0xFF3A3A4C),
  outlineVariant: Color(0xFF2A2A3C),
  scrim: Color(0x99000000),
  inverseSurface: Color(0xFFE8E8F0),
  onInverseSurface: Color(0xFF1C1C2E),
);
```

`background`, `onBackground` and `surfaceVariant` are deprecated in `ColorScheme`: use `surface`, `onSurface` and `surfaceContainerHighest`.

`outline` here is decorative. Controls that rely on their border to be seen need 3:1 against the surface, which WCAG 1.4.11 requires, so test yours.

## AppTypography extension with full copyWith and lerp

Use `TextTheme` first. An extension earns its place when a design has roles with no Material name, such as `caption` here. `copyWith` must copy every field, because a stub that returns `this` throws away whatever the caller asked to change.

```dart
// lib/core/theme/app_typography.dart
import 'package:material_ui/material_ui.dart';

@immutable
class AppTypography extends ThemeExtension<AppTypography> {
  const AppTypography({
    required this.displayLarge,
    required this.displayMedium,
    required this.headlineLarge,
    required this.headlineMedium,
    required this.titleLarge,
    required this.titleMedium,
    required this.bodyLarge,
    required this.bodyMedium,
    required this.labelLarge,
    required this.labelMedium,
    required this.caption,
  });

  factory AppTypography.of(Color color) => AppTypography(
        displayLarge: TextStyle(fontSize: 57, fontWeight: FontWeight.w400, color: color, height: 1.12),
        displayMedium: TextStyle(fontSize: 45, fontWeight: FontWeight.w400, color: color, height: 1.16),
        headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600, color: color, height: 1.25),
        headlineMedium: TextStyle(fontSize: 28, fontWeight: FontWeight.w600, color: color, height: 1.29),
        titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w600, color: color, height: 1.27),
        titleMedium: TextStyle(fontSize: 16, fontWeight: FontWeight.w600, color: color, height: 1.50),
        bodyLarge: TextStyle(fontSize: 16, fontWeight: FontWeight.w400, color: color, height: 1.50),
        bodyMedium: TextStyle(fontSize: 14, fontWeight: FontWeight.w400, color: color, height: 1.43),
        labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w600, color: color, height: 1.43),
        labelMedium: TextStyle(fontSize: 12, fontWeight: FontWeight.w500, color: color, height: 1.33),
        caption: TextStyle(fontSize: 11, fontWeight: FontWeight.w400, color: color, height: 1.45),
      );

  final TextStyle displayLarge;
  final TextStyle displayMedium;
  final TextStyle headlineLarge;
  final TextStyle headlineMedium;
  final TextStyle titleLarge;
  final TextStyle titleMedium;
  final TextStyle bodyLarge;
  final TextStyle bodyMedium;
  final TextStyle labelLarge;
  final TextStyle labelMedium;
  final TextStyle caption;

  @override
  AppTypography copyWith({
    TextStyle? displayLarge,
    TextStyle? displayMedium,
    TextStyle? headlineLarge,
    TextStyle? headlineMedium,
    TextStyle? titleLarge,
    TextStyle? titleMedium,
    TextStyle? bodyLarge,
    TextStyle? bodyMedium,
    TextStyle? labelLarge,
    TextStyle? labelMedium,
    TextStyle? caption,
  }) =>
      AppTypography(
        displayLarge: displayLarge ?? this.displayLarge,
        displayMedium: displayMedium ?? this.displayMedium,
        headlineLarge: headlineLarge ?? this.headlineLarge,
        headlineMedium: headlineMedium ?? this.headlineMedium,
        titleLarge: titleLarge ?? this.titleLarge,
        titleMedium: titleMedium ?? this.titleMedium,
        bodyLarge: bodyLarge ?? this.bodyLarge,
        bodyMedium: bodyMedium ?? this.bodyMedium,
        labelLarge: labelLarge ?? this.labelLarge,
        labelMedium: labelMedium ?? this.labelMedium,
        caption: caption ?? this.caption,
      );

  @override
  AppTypography lerp(AppTypography? other, double t) {
    if (other == null) return this;
    return AppTypography(
      displayLarge: TextStyle.lerp(displayLarge, other.displayLarge, t)!,
      displayMedium: TextStyle.lerp(displayMedium, other.displayMedium, t)!,
      headlineLarge: TextStyle.lerp(headlineLarge, other.headlineLarge, t)!,
      headlineMedium: TextStyle.lerp(headlineMedium, other.headlineMedium, t)!,
      titleLarge: TextStyle.lerp(titleLarge, other.titleLarge, t)!,
      titleMedium: TextStyle.lerp(titleMedium, other.titleMedium, t)!,
      bodyLarge: TextStyle.lerp(bodyLarge, other.bodyLarge, t)!,
      bodyMedium: TextStyle.lerp(bodyMedium, other.bodyMedium, t)!,
      labelLarge: TextStyle.lerp(labelLarge, other.labelLarge, t)!,
      labelMedium: TextStyle.lerp(labelMedium, other.labelMedium, t)!,
      caption: TextStyle.lerp(caption, other.caption, t)!,
    );
  }
}

extension AppTypographyX on BuildContext {
  AppTypography get appTypography => Theme.of(this).extension<AppTypography>()!;
}
```

Add it beside `AppColors`: `extensions: [AppColors.light, AppTypography.of(lightScheme.onSurface)]`.
