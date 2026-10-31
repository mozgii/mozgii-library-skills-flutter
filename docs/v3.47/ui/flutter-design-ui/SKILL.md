---
name: flutter-design-ui
description: Builds polished Flutter screens and components from theme tokens - Material 3 setup, responsive layouts with LayoutBuilder and Material breakpoints, glassmorphism and gradient cards, shimmer skeletons, CustomPainter shapes, cached images, dark mode rules, an adaptive navigation shell, and safe areas. Use when asked to build a new screen or custom widget, make a card look premium, add loading skeletons or frosted glass, fix a screen that breaks in dark mode, or set up navigation that adapts to phones, tablets and foldables.
metadata:
  tags:
  - flutter
  - ui
  - material3
  - design-system
  - responsive
  - shimmer
---
# Flutter Design and UI

Good Flutter UI is mostly discipline: every color from the theme, every size from a token, a skeleton where data will land, and a layout that holds from a 320-wide phone to a desktop window. This skill builds the screens and components on top of a theme; `flutter-theme` goes deeper into the theme itself.

## Step 0: Read the Project's Versions

Checked against Flutter 3.47.9, `material_ui` 1.9.0, `cupertino_ui` 1.1.5, `flex_color_scheme` 9.0.0, `dynamic_color` 2.1.0, `google_fonts` 9.0.0, `gap` 3.0.1, `shimmer` 4.0.0 and `cached_network_image` 4.0.6.

```bash
flutter --version
grep -nE "^\s+(material_ui|cupertino_ui|flex_color_scheme|shimmer|gap|cached_network_image):" pubspec.yaml
```

`shimmer` 4 and `cached_network_image` 4 depend on `material_ui`, so an app still importing `package:flutter/material.dart` ends up with two Material libraries whose types don't mix. Migrate with `dart fix --apply --code=migrate_design_widgets`. The examples also import `cupertino_ui: ^1.1.5`, `flutter_riverpod: ^3.4.3`, `go_router: ^18.1.2`, `material_ui: ^1.9.0`.

## Which Building Block to Use

| You need | Use | Why the others lose here |
|---|---|---|
| Layout that depends on the space a widget has | `LayoutBuilder` | `MediaQuery.sizeOf` gives the window, not the parent |
| Space between children | `Gap` with an `AppSpacing` token | `SizedBox` needs a direction; `Gap` follows the parent |
| Content on its way | A shimmer skeleton at the final size | A spinner tells you nothing about layout, and the page jumps |
| A frosted surface | `LiquidGlassCard` | A plain tinted box looks flat over images |
| A shape `BoxDecoration` can't do | `CustomPainter` | Stacking clipped widgets costs more and is hard to change |
| A photo from the network | `CachedNetworkImage` | `Image.network` keeps no disk cache |

## Design System Setup with Material 3

Colors live in `ColorScheme`, and `AppColors` adds what it lacks. The tokens used here (`AppColors`, `AppSpacing`, `AppRadius`) are in [references/design-tokens.md](references/design-tokens.md). `FlexColorScheme` builds the theme; `defaultRadius` sets every component's corner, and the per-component radii override it. There's no `buttonRadius`: use `defaultRadius`, or `elevatedButtonRadius`, `filledButtonRadius`, `outlinedButtonRadius` and `textButtonRadius`.

```yaml
dependencies:
  flex_color_scheme: ^9.0.0
  dynamic_color: ^2.1.0
  google_fonts: ^9.0.0
  gap: ^3.0.1
  shimmer: ^4.0.0
  cached_network_image: ^4.0.6
```

```dart
// lib/core/theme/app_theme.dart
import 'package:dynamic_color/dynamic_color.dart';
import 'package:flex_color_scheme/flex_color_scheme.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:material_ui/material_ui.dart';

import 'app_colors.dart';
import 'app_tokens.dart';

abstract final class AppTheme {
  static const _seed = Color(0xFF6B4EFF);
  static const _subThemes = FlexSubThemesData(
    interactionEffects: true,
    defaultRadius: AppRadius.md,
    cardRadius: AppRadius.lg,
    inputDecoratorRadius: AppRadius.md,
  );

  static ThemeData light({ColorScheme? dynamicScheme}) => FlexThemeData.light(
        colorScheme: dynamicScheme ?? ColorScheme.fromSeed(seedColor: _seed),
        fontFamily: GoogleFonts.inter().fontFamily,
        subThemesData: _subThemes,
        extensions: const [AppColors.light],
      );

  static ThemeData dark({ColorScheme? dynamicScheme}) => FlexThemeData.dark(
        colorScheme: dynamicScheme ??
            ColorScheme.fromSeed(seedColor: _seed, brightness: Brightness.dark),
        fontFamily: GoogleFonts.inter().fontFamily,
        subThemesData: _subThemes,
        extensions: const [AppColors.dark],
      );
}

class App extends StatelessWidget {
  const App({super.key, required this.router});

  final RouterConfig<Object> router;

  @override
  Widget build(BuildContext context) => DynamicColorBuilder(
        builder: (lightDynamic, darkDynamic) => MaterialApp.router(
          theme: AppTheme.light(dynamicScheme: lightDynamic),
          darkTheme: AppTheme.dark(dynamicScheme: darkDynamic),
          routerConfig: router,
        ),
      );
}
```

`DynamicColorBuilder` gives Android 12+ wallpaper colors, and `null` elsewhere, so the seed scheme stays the fallback.

## Spacing and Typography Rules

Never hard-code a size: use `AppSpacing`, with `Gap(AppSpacing.sm)` between widgets. Text uses only `Theme.of(context).textTheme` styles, such as `text.headlineMedium` and `text.bodyLarge`, so a person's text scale and the app's type ramp both apply.

## Worked Example: A Profile Card with Every State

This card has loading, error, empty-avatar and loaded states, takes its colors from the theme, holds its size through all of them so nothing jumps, and gives screen readers one sentence.

```dart
// lib/shared/widgets/profile_card.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:gap/gap.dart';
import 'package:material_ui/material_ui.dart';

import '../../core/theme/app_tokens.dart';
import 'avatar.dart';
import 'profile_card_skeleton.dart';

class Profile {
  const Profile({required this.name, required this.role, this.avatarUrl});

  final String name;
  final String role;
  final String? avatarUrl;
}

/// Yours: load the profile from your API.
final profileProvider = FutureProvider<Profile>(
  (ref) => throw UnimplementedError(),
);

class ProfileCard extends ConsumerWidget {
  const ProfileCard({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(AppSpacing.base),
        child: ref.watch(profileProvider).when(
              loading: () => const ProfileCardSkeleton(),
              error: (error, _) => TextButton(
                onPressed: () => ref.invalidate(profileProvider),
                child: const Text('Could not load the profile. Try again'),
              ),
              data: (profile) => _ProfileRow(profile: profile),
            ),
      ),
    );
  }
}

class _ProfileRow extends StatelessWidget {
  const _ProfileRow({required this.profile});

  final Profile profile;

  @override
  Widget build(BuildContext context) {
    final text = Theme.of(context).textTheme;
    return Semantics(
      container: true,
      label: '${profile.name}, ${profile.role}',
      excludeSemantics: true,
      child: Row(
        children: [
          Avatar(url: profile.avatarUrl),
          const Gap(AppSpacing.base),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(profile.name, style: text.titleMedium, maxLines: 1, overflow: TextOverflow.ellipsis),
                Text(profile.role, style: text.bodyMedium, maxLines: 1, overflow: TextOverflow.ellipsis),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

## Shimmer Loading States

Give an async widget a Shimmer skeleton at the size of the loaded content, so the page doesn't move when data lands. `Shimmer.fromColors` paints a moving gradient through the opaque pixels of its child, so a child's own color doesn't show: the `Colors.white` below is a mask, and the one hard-coded color this skill allows. `enabled: false` stops the sweep for anyone who asked for less motion.

```dart
// lib/shared/widgets/profile_card_skeleton.dart
import 'package:gap/gap.dart';
import 'package:material_ui/material_ui.dart';
import 'package:shimmer/shimmer.dart';

import '../../core/theme/app_tokens.dart';
import 'avatar.dart';

class ShimmerBox extends StatelessWidget {
  const ShimmerBox({super.key, this.width, this.height = 16, this.borderRadius = AppRadius.sm});

  final double? width;
  final double height;
  final double borderRadius;

  @override
  Widget build(BuildContext context) {
    final colors = Theme.of(context).colorScheme;
    return Shimmer.fromColors(
      baseColor: colors.surfaceContainerHighest,
      highlightColor: colors.surface,
      enabled: !MediaQuery.disableAnimationsOf(context),
      child: Container(
        width: width ?? double.infinity,
        height: height,
        decoration: BoxDecoration(
          color: Colors.white,
          borderRadius: BorderRadius.circular(borderRadius),
        ),
      ),
    );
  }
}

class ProfileCardSkeleton extends StatelessWidget {
  const ProfileCardSkeleton({super.key});

  @override
  Widget build(BuildContext context) => const Row(
        children: [
          ShimmerBox(width: Avatar.size, height: Avatar.size, borderRadius: AppRadius.xl),
          Gap(AppSpacing.base),
          Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              ShimmerBox(width: 120, height: 14),
              Gap(AppSpacing.xs),
              ShimmerBox(width: 80, height: 12),
            ],
          ),
        ],
      );
}
```

## Cached Image Loading

Use `CachedNetworkImage` for network images, and never `Image.network`. Set `memCacheWidth` to the display size times the device pixel ratio to limit decoded memory.

```dart
// lib/shared/widgets/avatar.dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:material_ui/material_ui.dart';

import '../../core/theme/app_tokens.dart';
import 'profile_card_skeleton.dart';

class Avatar extends StatelessWidget {
  const Avatar({super.key, required this.url});

  static const double size = 48;
  final String? url;

  @override
  Widget build(BuildContext context) {
    const fallback = CircleAvatar(radius: size / 2, child: Icon(Icons.person));
    final imageUrl = url;
    if (imageUrl == null) return fallback;
    final decode = (size * MediaQuery.devicePixelRatioOf(context)).round();
    return CachedNetworkImage(
      imageUrl: imageUrl,
      width: size,
      height: size,
      memCacheWidth: decode,
      memCacheHeight: decode,
      imageBuilder: (context, image) =>
          CircleAvatar(backgroundImage: image, radius: size / 2),
      placeholder: (context, url) =>
          const ShimmerBox(width: size, height: size, borderRadius: AppRadius.xl),
      errorWidget: (context, url, error) => fallback,
    );
  }
}
```

## Responsive Layouts with LayoutBuilder

Use `LayoutBuilder` for decisions about the space a widget has, and `MediaQuery.sizeOf` for the window; never `MediaQuery.of(context).size`, which rebuilds on every media change. Breakpoints follow Material's window size classes: compact below 600, medium from 600, expanded from 840. A layout wrapper and a grid are in [references/responsive-layouts.md](references/responsive-layouts.md), and the navigation shell that changes form with width is in [references/adaptive-shell.md](references/adaptive-shell.md).

## Glassmorphism and Gradient Cards

Glassmorphism is a frosted surface: a blur of what's behind, a translucent fill and a thin light border. `LiquidGlassCard` draws it with `BackdropFilter` on every platform. It's an approximation of Apple's Liquid Glass, not the real material, so there's no version check and no native path to get wrong. For the iOS 26 material and how to read the iOS version, see `flutter-theme`.

Every `BackdropFilter` reads back what's behind it and blurs it on the raster thread, and that work is Flutter's on every platform: on iOS it runs in Impeller, not in Core Animation, so iOS gets no free pass. Keep to 2 or 3 per frame on Android. Group several with `BackdropGroup` and `BackdropFilter.grouped` to share one backdrop.

The card's code is in `references/design-tokens.md`.

`GradientCard`, `CustomPainter` shapes and the dark mode rules are in [references/painters-and-dark-mode.md](references/painters-and-dark-mode.md).

## Minimum Touch Target

Every tappable element needs a 48 × 48 hit area, whatever it draws. Material buttons already have it. For custom tappables use `MinTouchTarget` from [references/design-tokens.md](references/design-tokens.md), and for an `IconButton`, set `minimumSize` and `tapTargetSize` in its style: `IconButton.styleFrom(minimumSize: const Size(48, 48), tapTargetSize: MaterialTapTargetSize.padded)`.

## Verify

The examples were compiled, not run on a device, so check the look and motion on real hardware.

```bash
flutter analyze
flutter test
```

Pump each state and check it holds up at a phone width, with the largest text, and against the accessibility guidelines:

```dart
// test/shared/profile_card_test.dart
import 'package:app/core/theme/app_colors.dart';
import 'package:app/shared/widgets/profile_card_skeleton.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  testWidgets('skeleton keeps its layout at 320 wide', (tester) async {
    tester.view.physicalSize = const Size(320, 640);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.reset);
    final handle = tester.ensureSemantics();

    await tester.pumpWidget(
      MaterialApp(
        theme: ThemeData(extensions: const [AppColors.light]),
        home: const Scaffold(body: ProfileCardSkeleton()),
      ),
    );

    expect(tester.takeException(), isNull);
    await expectLater(tester, meetsGuideline(textContrastGuideline));
    handle.dispose();
  });
}
```

Then check the real thing by eye in light and dark, at 320 wide and on a tablet, with the system text size at its largest. Use `flutter run --profile` for any screen with blur.

## Gotchas

1. **`The named parameter 'buttonRadius' isn't defined`.** Symptom: a copied `FlexSubThemesData` fails. Cause: there's no such field. Fix: `defaultRadius`, or the per-button radii.
2. **`The name 'Theme' is defined in the libraries...`.** Symptom: everything turns ambiguous after adding `shimmer` 4. Cause: mixed SDK and `material_ui` imports. Fix: run the `dart fix` from Step 0.
3. **`Null check operator used on a null value` from `extension<AppColors>()!`.** Symptom: a red screen in one mode. Cause: that `ThemeData` lacks the extension. Fix: pass it to every theme.
4. **`dart:io` error on web.** Symptom: `Unsupported operation: Platform._operatingSystem`. Cause: `Platform.isIOS` in UI code. Fix: `defaultTargetPlatform`.
5. **Page jumps when data arrives.** Symptom: layout shift after loading. Cause: the skeleton is a different size from the content. Fix: share a size constant, as `Avatar.size` does.
6. **Janky scroll with glass cards on Android.** Symptom: the raster phase overruns in DevTools. Cause: too many `BackdropFilter`s. Fix: `BackdropGroup`, or drop the blur inside lists.
7. **`A RenderFlex overflowed by N pixels` at large text.** Symptom: yellow and black stripes. Cause: a fixed-height row. Fix: `Expanded` with `maxLines` and `overflow`.
8. **`google_fonts was unable to load font` in a test.** Symptom: a font error in test output. Cause: tests have no network. Fix: test with a plain `ThemeData` that carries your extensions, or bundle the font.

## Anti-patterns

| Don't | Do instead |
|---|---|
| `Colors.white` on a surface | `colorScheme.surface`, which adapts to dark mode |
| Hard-code `16` or `Color(0xFF...)` in a widget | Tokens |
| `SizedBox` for gaps in a `Column` | `Gap` |
| Spinner for a known layout | A skeleton at the final size |
| `MediaQuery.of(context).size` | `MediaQuery.sizeOf(context)` or `LayoutBuilder` |
| `Platform.isIOS` in a leaf widget | Resolve it once in the shell with `defaultTargetPlatform` |

## Pre-merge Checklist

- [ ] Colors from the theme, sizes from tokens
- [ ] Loading, error, empty and loaded states all shown
- [ ] Skeletons match the final layout
- [ ] Reviewed in light and dark
- [ ] Holds from 320 wide to a large window, and at the largest text size
- [ ] Touch targets at least 48, with semantic labels
- [ ] Images decode at display size
- [ ] Blur checked in profile mode

## See Also

- `flutter-theme` for tokens, high contrast and fonts; it defines the same `AppColors`, `LiquidGlassCard` and `MinTouchTarget` as this skill, so the two combine
- `flutter-build-responsive-layout` for breakpoints in depth
- `flutter-animations` for motion
- `flutter-state` for a persisted theme mode toggle
- `flutter-fix-layout-issues` for overflow errors
