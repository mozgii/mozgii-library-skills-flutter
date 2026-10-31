---
title: Flutter adaptive components, typography and folds
author: MOZGII, Inc.
lang: en
tags: [flutter, theme, adaptive, typography, foldable]
description: When to use Cupertino or Material widgets, an adaptive dialog, platform fonts, a fold-aware two-pane layout.
---
# Flutter Adaptive Components, Typography and Folds

A shared theme stops at the platform line: iOS and Android people expect different controls. Resolve the difference once, in one widget, and let screens use that.

## Cupertino or Material per component

Prefer the `.adaptive` constructor when one exists. It keeps one call site, and the framework picks the widget.

| Situation | iOS, macOS | Android, others | One call |
|---|---|---|---|
| Primary navigation | `CupertinoTabBar` | `NavigationBar` | None; see `flutter-design-ui` |
| Alert or dialog | `CupertinoAlertDialog` | `AlertDialog` | `AlertDialog.adaptive` |
| Toggle | `CupertinoSwitch` | `Switch` | `Switch.adaptive` |
| Activity indicator | `CupertinoActivityIndicator` | `CircularProgressIndicator` | `CircularProgressIndicator.adaptive` |
| Text field | `CupertinoTextField` | `TextField` | None |
| Date or time picker | `CupertinoDatePicker` | `CalendarDatePicker` | None |
| Scroll physics | `BouncingScrollPhysics` | `ClampingScrollPhysics` | Applied by the platform's `ScrollBehavior` |

```dart
import 'package:material_ui/material_ui.dart';

Widget savingRow({
  required bool saving,
  required bool enabled,
  required ValueChanged<bool> onChanged,
  required Color brand,
}) =>
    Row(
      children: [
        Switch.adaptive(
          value: enabled,
          onChanged: onChanged,
          activeTrackColor: brand,
        ),
        if (saving) const CircularProgressIndicator.adaptive(),
      ],
    );
```

## Adaptive alert dialog

`showAdaptiveDialog` uses the platform's route, and `AlertDialog.adaptive` the platform's look. Only the actions differ, so a small data class carries them and the builder renders each one for the platform.

```dart
import 'package:cupertino_ui/cupertino_ui.dart';
import 'package:flutter/foundation.dart';
import 'package:material_ui/material_ui.dart';

class AdaptiveDialogAction {
  const AdaptiveDialogAction({
    required this.label,
    required this.onPressed,
    this.isDestructive = false,
  });

  final String label;
  final VoidCallback? onPressed;
  final bool isDestructive;
}

Future<T?> showAdaptiveAlert<T>({
  required BuildContext context,
  required String title,
  required String content,
  required List<AdaptiveDialogAction> actions,
}) {
  final colors = Theme.of(context).colorScheme;
  final isApple = defaultTargetPlatform == TargetPlatform.iOS ||
      defaultTargetPlatform == TargetPlatform.macOS;
  return showAdaptiveDialog<T>(
    context: context,
    builder: (_) => AlertDialog.adaptive(
      title: Text(title),
      content: Text(content),
      actions: [
        for (final action in actions)
          isApple
              ? CupertinoDialogAction(
                  isDestructiveAction: action.isDestructive,
                  onPressed: action.onPressed,
                  child: Text(action.label),
                )
              : TextButton(
                  onPressed: action.onPressed,
                  style: action.isDestructive
                      ? TextButton.styleFrom(foregroundColor: colors.error)
                      : null,
                  child: Text(action.label),
                ),
      ],
    ),
  );
}
```

## Platform-adaptive typography

Apple platforms keep the system font and everything else gets the brand face, here Inter. The framework's own Apple text themes use `CupertinoSystemText` and `CupertinoSystemDisplay`, which it resolves to San Francisco, so leave `fontFamily` unset there. `.SF Pro Text` isn't a name to hard-code. `google_fonts` downloads on first use: bundle the fonts and set `GoogleFonts.config.allowRuntimeFetching = false` before a release that has to work offline.

```dart
import 'package:flutter/foundation.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:material_ui/material_ui.dart';

abstract final class AppFonts {
  static TextTheme buildTextTheme(ColorScheme colorScheme) {
    final base = _baseTextTheme(colorScheme);
    final isApple = defaultTargetPlatform == TargetPlatform.iOS ||
        defaultTargetPlatform == TargetPlatform.macOS;
    return isApple ? base : GoogleFonts.interTextTheme(base);
  }

  static TextTheme _baseTextTheme(ColorScheme colorScheme) {
    final color = colorScheme.onSurface;
    return TextTheme(
      displayLarge: TextStyle(fontSize: 57, fontWeight: FontWeight.w400, color: color, letterSpacing: -0.25),
      displayMedium: TextStyle(fontSize: 45, fontWeight: FontWeight.w400, color: color),
      displaySmall: TextStyle(fontSize: 36, fontWeight: FontWeight.w400, color: color),
      headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600, color: color),
      headlineMedium: TextStyle(fontSize: 28, fontWeight: FontWeight.w600, color: color),
      headlineSmall: TextStyle(fontSize: 24, fontWeight: FontWeight.w600, color: color),
      titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w600, color: color),
      titleMedium: TextStyle(fontSize: 16, fontWeight: FontWeight.w600, color: color, letterSpacing: 0.15),
      titleSmall: TextStyle(fontSize: 14, fontWeight: FontWeight.w600, color: color, letterSpacing: 0.1),
      bodyLarge: TextStyle(fontSize: 16, fontWeight: FontWeight.w400, color: color, letterSpacing: 0.5),
      bodyMedium: TextStyle(fontSize: 14, fontWeight: FontWeight.w400, color: color, letterSpacing: 0.25),
      bodySmall: TextStyle(fontSize: 12, fontWeight: FontWeight.w400, color: color, letterSpacing: 0.4),
      labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w600, color: color, letterSpacing: 0.1),
      labelMedium: TextStyle(fontSize: 12, fontWeight: FontWeight.w500, color: color, letterSpacing: 0.5),
      labelSmall: TextStyle(fontSize: 11, fontWeight: FontWeight.w500, color: color, letterSpacing: 0.5),
    );
  }
}
```

Don't scale text by hand. `MediaQuery.textScalerOf` applies the person's setting to every `Text` already.

## Fold-aware two-pane layout

`MediaQuery.displayFeaturesOf` reports folds and hinges with their position. Only a feature that splits the window left from right helps a side-by-side layout, so a horizontal hinge on a flip phone is ignored. Place the panes by the feature's real bounds, because a hinge isn't always in the middle. A `fold` has zero width, a `hinge` has real width, and the gap below handles both.

```dart
import 'dart:ui' show DisplayFeature, DisplayFeatureType;

import 'package:material_ui/material_ui.dart';

class FoldAwareLayout extends StatelessWidget {
  const FoldAwareLayout({
    super.key,
    required this.primary,
    required this.secondary,
    this.breakpoint = 600,
  });

  final Widget primary;
  final Widget secondary;
  final double breakpoint;

  static bool _splitsSideBySide(DisplayFeature feature) =>
      feature.type != DisplayFeatureType.cutout &&
      feature.bounds.height >= feature.bounds.width;

  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.sizeOf(context).width;
    final split = MediaQuery.displayFeaturesOf(context)
        .where(_splitsSideBySide)
        .firstOrNull;

    if (split != null) {
      return Row(
        children: [
          SizedBox(width: split.bounds.left, child: primary),
          SizedBox(width: split.bounds.width),
          Expanded(child: secondary),
        ],
      );
    }
    if (width >= breakpoint) {
      return Row(
        children: [
          Expanded(child: primary),
          Expanded(child: secondary),
        ],
      );
    }
    return primary;
  }
}
```

## MinTouchTarget: 48 Pixel Tap Targets

`MinTouchTarget` keeps a small control's look and gives it a tappable area of at least 48 by 48 pixels, the size the tap target guidelines check.

```dart
// lib/core/theme/min_touch_target.dart
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
