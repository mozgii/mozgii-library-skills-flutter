---
title: Flutter adaptive navigation shell
author: MOZGII, Inc.
lang: en
tags: [flutter, ui, navigation, adaptive, foldable]
description: One AdaptiveScaffold for phones, tablets, foldables, iOS and Android, go_router wiring, platform colors and edge-to-edge headers.
---
# Flutter Adaptive Navigation Shell

Navigation changes form with the window and the platform. One widget decides, once, at the top of the app, and no screen below has to care.

## AdaptiveScaffold: one shell for every window

`AdaptiveScaffold` shows a `NavigationRail` at 600 wide and up, for tablets, unfolded foldables and desktops. Below that, iOS gets a `CupertinoTabBar` and everything else a Material `NavigationBar`. Check with `defaultTargetPlatform`, since `Platform.isIOS` throws on web. Keep this check at shell level, never in leaf widgets.

```dart
import 'package:cupertino_ui/cupertino_ui.dart';
import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';

class AdaptiveTab {
  const AdaptiveTab({
    required this.icon,
    required this.selectedIcon,
    required this.label,
  });

  final IconData icon;
  final IconData selectedIcon;
  final String label;
}

class AdaptiveScaffold extends StatelessWidget {
  const AdaptiveScaffold({
    super.key,
    required this.body,
    required this.tabs,
    required this.selectedIndex,
    required this.onDestinationSelected,
  });

  static const double _railBreakpoint = 600;

  final Widget body;
  final List<AdaptiveTab> tabs;
  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;

  @override
  Widget build(BuildContext context) => LayoutBuilder(
        builder: (context, constraints) {
          if (constraints.maxWidth >= _railBreakpoint) {
            return Scaffold(body: Row(children: [_rail(), const VerticalDivider(thickness: 1, width: 1), Expanded(child: body)]));
          }
          return Scaffold(
            body: body,
            bottomNavigationBar: defaultTargetPlatform == TargetPlatform.iOS
                ? _cupertinoBar()
                : _materialBar(),
          );
        },
      );

  Widget _rail() => NavigationRail(
        selectedIndex: selectedIndex,
        onDestinationSelected: onDestinationSelected,
        labelType: NavigationRailLabelType.selected,
        destinations: [
          for (final tab in tabs)
            NavigationRailDestination(
              icon: Icon(tab.icon),
              selectedIcon: Icon(tab.selectedIcon),
              label: Text(tab.label),
            ),
        ],
      );

  Widget _materialBar() => NavigationBar(
        selectedIndex: selectedIndex,
        onDestinationSelected: onDestinationSelected,
        destinations: [
          for (final tab in tabs)
            NavigationDestination(
              icon: Icon(tab.icon),
              selectedIcon: Icon(tab.selectedIcon),
              label: tab.label,
            ),
        ],
      );

  Widget _cupertinoBar() => CupertinoTabBar(
        currentIndex: selectedIndex,
        onTap: onDestinationSelected,
        items: [
          for (final tab in tabs)
            BottomNavigationBarItem(
              icon: Icon(tab.icon),
              activeIcon: Icon(tab.selectedIcon),
              label: tab.label,
            ),
        ],
      );
}

/// Wiring for go_router's StatefulShellRoute.indexedStack builder.
Widget shellBuilder(
  StatefulNavigationShell shell,
  List<AdaptiveTab> tabs,
) =>
    AdaptiveScaffold(
      body: shell,
      tabs: tabs,
      selectedIndex: shell.currentIndex,
      onDestinationSelected: shell.goBranch,
    );
```

## Foldable hinge rule

Read `MediaQuery.displayFeaturesOf(context)` to find a hinge or fold, where each `DisplayFeature` has a `type` and `bounds`. In book posture, keep a `FloatingActionButton` and swipe-sensitive widgets off the fold line, and pad with the feature's `bounds` to push content clear of a hinge. Dialogs already avoid features through `DisplayFeatureSubScreen`, and you can use that widget for your own popups.

## Platform colors with CupertinoColors

Prefer `colorScheme` wherever it covers the need. Reach for `CupertinoColors` only for iOS system colors with no Material match, such as a system background or separator lines. `resolveFrom` applies dark mode and contrast.

```dart
import 'package:cupertino_ui/cupertino_ui.dart';
import 'package:flutter/foundation.dart';
import 'package:material_ui/material_ui.dart';

extension PlatformColorX on BuildContext {
  bool get _isIOS => defaultTargetPlatform == TargetPlatform.iOS;

  Color get adaptiveBackground => _isIOS
      ? CupertinoColors.systemBackground.resolveFrom(this)
      : Theme.of(this).colorScheme.surface;

  Color get adaptiveSeparator => _isIOS
      ? CupertinoColors.separator.resolveFrom(this)
      : Theme.of(this).colorScheme.outlineVariant;

  Color get adaptivePrimaryLabel => _isIOS
      ? CupertinoColors.label.resolveFrom(this)
      : Theme.of(this).colorScheme.onSurface;
}
```

## Dynamic Island, notch and edge-to-edge headers

`SafeArea` handles the notch, the Dynamic Island and the home indicator on its own, so a standard layout needs no custom padding. A header that runs behind the status bar reads the top inset itself.

```dart
import 'package:material_ui/material_ui.dart';

class EdgeToEdgeHeader extends StatelessWidget {
  const EdgeToEdgeHeader({super.key});

  @override
  Widget build(BuildContext context) {
    final topInset = MediaQuery.paddingOf(context).top;
    final colors = Theme.of(context).colorScheme;
    return Container(
      padding: EdgeInsets.only(top: topInset + 16, left: 16, right: 16, bottom: 16),
      color: colors.primaryContainer,
      child: Text(
        'Custom Header',
        style: TextStyle(color: colors.onPrimaryContainer),
      ),
    );
  }
}
```
