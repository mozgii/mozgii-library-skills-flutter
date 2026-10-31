---
name: flutter-theme
description: Builds a production Flutter theme - ThemeExtension design tokens, FlexColorScheme ThemeData, dynamic color, dark and high-contrast modes, tested WCAG contrast, typography, platform-adaptive components, touch targets, foldable layouts, and an honest Liquid Glass look. Use when creating or updating an app theme, replacing hard-coded colors and spacing with tokens, adding dark mode or high contrast, fixing contrast failures, or adding a Liquid Glass effect to an existing app.
metadata:
  tags:
  - flutter
  - theme
  - design-system
  - liquid-glass
  - material3
  - cupertino
  - foldable
  - dark-mode
---
# Flutter Theme

A theme is the one place an app's colors, type, shape and spacing are decided. Widgets read values from it, so a rebrand or a dark mode changes one file instead of a hundred. Values go in the theme, never inline.

## Step 0: Read the Project's Versions

Checked against Flutter 3.47.9, `material_ui` 1.9.0, `cupertino_ui` 1.1.5, `flex_color_scheme` 9.0.0, `dynamic_color` 2.1.0 and `google_fonts` 9.0.0.

```bash
flutter --version
grep -nE "^\s+(material_ui|cupertino_ui|flex_color_scheme|dynamic_color|google_fonts):" pubspec.yaml
```

Every example imports `package:material_ui/material_ui.dart`. Types from it and from `package:flutter/material.dart` don't mix, so migrate first with `dart fix --apply --code=migrate_design_widgets`. `flex_color_scheme` 9 and `dynamic_color` 2 already build on `material_ui`.

Skip this skill if the project already has a complete token system, or a white-label SDK owns the theme. The examples also import `cupertino_ui: ^1.1.5`, `device_info_plus: ^13.3.0`, `dynamic_color: ^2.1.0`, `flex_color_scheme: ^9.0.0`, `flutter_riverpod: ^3.4.3`, `google_fonts: ^9.0.0`, `material_ui: ^1.9.0`.

## Where Each Value Belongs

| Value | Put it in | Why the others lose here |
|---|---|---|
| Brand and surface colors | `ColorScheme` | Every Material widget reads it; a parallel token set drifts |
| Success, warning, glass | `AppColors`, a `ThemeExtension` | `ColorScheme` has no such roles |
| Text styles | `TextTheme` | Widgets inherit it; use `AppTypography` only for roles it lacks |
| Spacing, radius, elevation | `abstract final class` constants | They don't change with brightness, so `lerp` adds nothing |
| Component shape and states | `FlexSubThemesData` | Hand-writing each component theme takes hundreds of lines |

## Design Tokens with ThemeExtension

`ThemeExtension` adds your own typed values to `ThemeData`. Because it implements `lerp`, a theme change animates these colors with everything else. Keep the class to what `ColorScheme` can't express.

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

`AppSpacing`, `AppElevation` and `AppRadius` constants, the full `ColorScheme` palette and an `AppTypography` extension with a complete `copyWith` and `lerp` are in [references/tokens.md](references/tokens.md).

## Worked Example: ThemeData from FlexColorScheme

`FlexThemeData` builds a complete `ThemeData` from a scheme and `FlexSubThemesData`. A `colorScheme` you pass wins over `scheme`, so dynamic color falls back to the brand scheme when it's unavailable. Both brightnesses share one sub-theme value, which keeps them from drifting. High contrast gets its own themes, because `MaterialApp` picks them only when the OS asks.

```dart
// lib/core/theme/app_theme.dart
import 'package:dynamic_color/dynamic_color.dart';
import 'package:flex_color_scheme/flex_color_scheme.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import 'app_colors.dart';
import 'app_tokens.dart';

abstract final class AppTheme {
  static const _subThemes = FlexSubThemesData(
    useInputDecoratorThemeInDialogs: true,
    cardRadius: AppRadius.lg,
    inputDecoratorRadius: AppRadius.md,
    inputDecoratorBorderType: FlexInputBorderType.outline,
    inputDecoratorUnfocusedHasBorder: true,
    inputDecoratorFocusedBorderWidth: 2,
    elevatedButtonRadius: AppRadius.full,
    elevatedButtonElevation: AppElevation.sm,
    filledButtonRadius: AppRadius.full,
    outlinedButtonRadius: AppRadius.full,
    textButtonRadius: AppRadius.full,
    navigationBarSelectedLabelSchemeColor: SchemeColor.primary,
    navigationBarIndicatorSchemeColor: SchemeColor.primaryContainer,
    navigationBarBackgroundSchemeColor: SchemeColor.surface,
    appBarBackgroundSchemeColor: SchemeColor.surface,
    appBarScrolledUnderElevation: 1,
    dialogRadius: AppRadius.xl,
    drawerRadius: 0,
  );

  static ThemeData light({ColorScheme? dynamicScheme}) => FlexThemeData.light(
        colorScheme: dynamicScheme,
        scheme: FlexScheme.indigo,
        surfaceMode: FlexSurfaceMode.levelSurfacesLowScaffold,
        blendLevel: 7,
        subThemesData: _subThemes,
        visualDensity: FlexColorScheme.comfortablePlatformDensity,
        extensions: const [AppColors.light],
      );

  static ThemeData dark({ColorScheme? dynamicScheme}) => FlexThemeData.dark(
        colorScheme: dynamicScheme,
        scheme: FlexScheme.indigo,
        surfaceMode: FlexSurfaceMode.levelSurfacesLowScaffold,
        blendLevel: 13,
        subThemesData: _subThemes,
        visualDensity: FlexColorScheme.comfortablePlatformDensity,
        extensions: const [AppColors.dark],
      );

  static ThemeData highContrast() => ThemeData(
        colorScheme: const ColorScheme.highContrastLight(
          primary: Color(0xFF0000EE),
          secondary: Color(0xFF4B0082),
          error: Color(0xFFA80000),
        ),
        extensions: const [AppColors.highContrast],
      );
}

final themeModeProvider =
    NotifierProvider<ThemeModeNotifier, ThemeMode>(ThemeModeNotifier.new);

class ThemeModeNotifier extends Notifier<ThemeMode> {
  @override
  ThemeMode build() => ThemeMode.system;

  void setMode(ThemeMode mode) => state = mode;
}

class ThemedApp extends ConsumerWidget {
  const ThemedApp({super.key, required this.router});

  final RouterConfig<Object> router;

  @override
  Widget build(BuildContext context, WidgetRef ref) => DynamicColorBuilder(
        builder: (lightDynamic, darkDynamic) => MaterialApp.router(
          theme: AppTheme.light(dynamicScheme: lightDynamic),
          darkTheme: AppTheme.dark(dynamicScheme: darkDynamic),
          highContrastTheme: AppTheme.highContrast(),
          themeMode: ref.watch(themeModeProvider),
          routerConfig: router,
        ),
      );
}
```

`DynamicColorBuilder` hands you `null` until the platform answers, and always on platforms without dynamic color, so the fallback has to look right. This is Android 12's wallpaper color. To keep the choice across launches, persist `ThemeMode` in `ThemeModeNotifier` with your storage layer.

## Dark Mode and High Contrast Settings

`MediaQuery.highContrastOf(context)` follows iOS Increase Contrast and, on Android, a contrast setting above standard. You rarely read it: `MaterialApp` swaps to `highContrastTheme` or `highContrastDarkTheme` by itself. `AppColors.highContrast` makes glass opaque on purpose, since a blurred backdrop is what that setting exists to remove. An app with a dark theme should set `highContrastDarkTheme` too; `ColorScheme.highContrastDark` is the starting point.

Don't use opacity on a semantic color for a disabled state unless no disabled token exists; then `onSurface` at alpha 0.38 is the Material value.

## Contrast Checks in Code

WCAG AA asks for 4.5:1 on normal text and 3:1 on large text; AAA asks for 7:1 and 4.5:1. `Color.computeLuminance` already implements the WCAG formula, so the check is short. Run it on every `onX` and `X` pair before a token ships.

```dart
// lib/core/theme/contrast_ratio.dart
import 'package:material_ui/material_ui.dart';

abstract final class ContrastRatio {
  static double calculate(Color foreground, Color background) {
    final a = foreground.computeLuminance();
    final b = background.computeLuminance();
    final lighter = a > b ? a : b;
    final darker = a > b ? b : a;
    return (lighter + 0.05) / (darker + 0.05);
  }

  static bool passesAA(Color fg, Color bg, {bool largeText = false}) =>
      calculate(fg, bg) >= (largeText ? 3.0 : 4.5);

  static bool passesAAA(Color fg, Color bg, {bool largeText = false}) =>
      calculate(fg, bg) >= (largeText ? 4.5 : 7.0);
}
```

## Touch Target Size per Platform

Use 48 as the cross-platform minimum: it satisfies every standard below with no per-platform logic.

| Source | Minimum |
|---|---|
| Material Design 3 (`kMinInteractiveDimension`) | 48 × 48 dp |
| Apple Human Interface Guidelines | 44 × 44 pt |
| WCAG 2.5.5 (AAA) | 44 × 44 CSS px |
| WCAG 2.5.8 (AA) | 24 × 24 CSS px |

A smaller control can still pass, because the touch target isn't the drawn size. Material 3 buttons draw at height 40 and `MaterialTapTargetSize.padded` grows the target to 48 on mobile; desktop platforms default to `shrinkWrap`. So the widget's visual size and its target are two separate numbers.

| Component | Default visual size | Touch target on mobile |
|---|---|---|
| `ElevatedButton`, `FilledButton`, `OutlinedButton`, `TextButton` | height 40, min width 64 | 48 tall |
| `IconButton` | 40 × 40 | 48 × 48 |
| `FloatingActionButton` | 56 × 56 | 56 × 56 |
| `FloatingActionButton.small` | 40 × 40 | 48 × 48 |
| `Chip` | height 32 | 48 tall |

On a 320-wide phone a 48 button is about 15% of the width, and about 11% on a 430-wide one, so keep to two buttons per row at most and use full-width buttons in forms. Icon-only buttons need a `tooltip`, which also gives a screen reader its label.

For a custom tappable, `MinTouchTarget` grows the hit area without changing what's drawn:

`MinTouchTarget` grows the tappable area to 48 pixels and is in `references/adaptive-components.md`.

## Apple Liquid Glass in Flutter

Liquid Glass is the material Apple introduced with iOS 26: specular highlights, depth-aware blur, tinting from what's behind it, and light that bends. `material_ui` 1.9.0 and `cupertino_ui` 1.1.5 ship no Liquid Glass widget, and Flutter draws its own pixels, so it can't reach UIKit's `UIGlassEffect`. `BackdropFilter` gives a look close enough for most apps. Lacking the real vibrancy processing, it's an approximation: say so in your design notes.

`LiquidGlassCard`, the native option, and how to read the iOS version are in [references/liquid-glass.md](references/liquid-glass.md).

## Platform-Adaptive Components

Flutter's `.adaptive` constructors are the first choice: `Switch.adaptive`, `CircularProgressIndicator.adaptive`, `AlertDialog.adaptive`, `showAdaptiveDialog`. Check the platform with `defaultTargetPlatform`, because `dart:io`'s `Platform` throws on web. The full table, an adaptive dialog, typography per platform and fold-aware panes are in [references/adaptive-components.md](references/adaptive-components.md).

## Verify

```bash
flutter analyze
flutter test
```

This test fails if any token pair drops below AA, or if a screen breaks tap target or contrast guidelines. Add a screen of your own to the second test. The examples were compiled, not run on a device, so check dynamic color on real hardware.

```dart
// test/core/theme/theme_test.dart
import 'package:app/core/theme/app_colors.dart';
import 'package:app/core/theme/app_theme.dart';
import 'package:app/core/theme/contrast_ratio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  test('every token pair meets WCAG AA', () {
    const sets = [AppColors.light, AppColors.dark, AppColors.highContrast];
    for (final colors in sets) {
      expect(ContrastRatio.passesAA(colors.onSuccess, colors.success), isTrue);
      expect(ContrastRatio.passesAA(colors.onWarning, colors.warning), isTrue);
    }
  });

  testWidgets('a themed screen meets accessibility guidelines', (tester) async {
    final handle = tester.ensureSemantics();
    await tester.pumpWidget(
      MaterialApp(
        theme: AppTheme.light(),
        home: Scaffold(
          body: Center(
            child: FilledButton(onPressed: () {}, child: const Text('Save')),
          ),
        ),
      ),
    );
    await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
    await expectLater(tester, meetsGuideline(iOSTapTargetGuideline));
    await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
    await expectLater(tester, meetsGuideline(textContrastGuideline));
    handle.dispose();
  });
}
```

Then check light, dark and high contrast at the largest text size on real iOS and Android devices.

## Gotchas

1. **`The named parameter 'useM3StyleDividerInM2' isn't defined`.** Symptom: a copied `FlexSubThemesData` fails. Cause: that field and `bottomNavigationBarUnselectedAlphaBlend` don't exist in 9.0.0. Fix: delete them; there are no replacements to find.
2. **`The name 'Theme' is defined in the libraries...`.** Symptom: ambiguous imports after adding `material_ui`. Cause: a file imports both it and the SDK's `material.dart`. Fix: run the `dart fix` above.
3. **`Null check operator used on a null value` at `extension<AppColors>()!`.** Symptom: a red screen on one theme only. Cause: one `ThemeData` lacks the extension, usually the high-contrast one. Fix: pass `extensions` to every theme.
4. **Warning badge fails an audit.** Symptom: white text on orange reads poorly. Cause: `#E67E22` with white is 2.85:1. Fix: `#B45309` is 5.02:1, or use dark text.
5. **`'activeColor' is deprecated`.** Symptom: an analyzer warning on `CupertinoSwitch`. Cause: it was renamed. Fix: `activeTrackColor`, or use `Switch.adaptive`.
6. **`'red' is deprecated`.** Symptom: an analyzer warning in contrast code. Cause: `Color` moved to floating point components. Fix: `.r`, `.g`, `.b` run from 0 to 1, or call `computeLuminance`.
7. **Theme animation jumps for custom tokens.** Symptom: tokens snap while the rest fades. Cause: `lerp` returned `this`. Fix: implement every field.
8. **Text on glass is unreadable.** Symptom: fine over one image and gone over another. Cause: low fill alpha. Fix: raise `glassFill` until body text passes over the brightest and darkest backdrops.

## Anti-patterns

| Don't | Do instead |
|---|---|
| `Colors.red` or `Color(0x...)` in a widget | `context.appColors.warning`, `colorScheme.error` |
| Mirror `ColorScheme` in `AppColors` | Add only roles it doesn't have |
| Return `this` from `copyWith` | Copy every field |
| `Platform.isIOS` in a shared widget | `defaultTargetPlatform` |
| Set `textScaleFactor` by hand | Leave scaling to `MediaQuery.textScalerOf` |
| Ship a stub that claims native glass | Name it an approximation, or build the bridge |

## Pre-merge Checklist

- [ ] No hard-coded colors or sizes in widgets
- [ ] Every `ThemeData` carries every extension
- [ ] `copyWith` and `lerp` cover every field
- [ ] Contrast test passes for light, dark and high contrast
- [ ] `androidTapTargetGuideline` and `iOSTapTargetGuideline` pass on changed screens
- [ ] Checked with and without dynamic color, and at the largest text size

## See Also

- `flutter-design-ui` for screens, cards, skeletons and the navigation shell; it uses the same `AppColors`, `AppSpacing`, `LiquidGlassCard` and `MinTouchTarget`
- `flutter-build-responsive-layout` for breakpoints and panes
- `flutter-state` for persisting the theme mode
- `flutter-animations` for motion
