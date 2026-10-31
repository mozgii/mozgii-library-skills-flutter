---
title: Flutter responsive layout widgets
author: MOZGII, Inc.
lang: en
tags: [flutter, ui, responsive, layout]
description: A breakpoint wrapper on LayoutBuilder and an adaptive grid that needs no breakpoints.
---
# Flutter Responsive Layout Widgets

A layout should answer to the space it has. These two widgets cover most screens: one swaps its child at Material's window size classes, and one lets the column count follow the width.

Use `LayoutBuilder` for decisions about the space a widget has, and `MediaQuery.sizeOf` for the window; never `MediaQuery.of(context).size`, which rebuilds on every media change. Breakpoints follow Material's window size classes: compact below 600, medium from 600, expanded from 840.

```dart
// lib/shared/widgets/responsive_layout.dart
import 'package:material_ui/material_ui.dart';

import '../../core/theme/app_tokens.dart';

abstract final class Breakpoints {
  static const double medium = 600;
  static const double expanded = 840;
}

class ResponsiveLayout extends StatelessWidget {
  const ResponsiveLayout({super.key, required this.compact, this.medium, this.expanded});

  final Widget compact;
  final Widget? medium;
  final Widget? expanded;

  @override
  Widget build(BuildContext context) => LayoutBuilder(
        builder: (context, constraints) {
          final width = constraints.maxWidth;
          if (width >= Breakpoints.expanded) return expanded ?? medium ?? compact;
          if (width >= Breakpoints.medium) return medium ?? compact;
          return compact;
        },
      );
}

class AdaptiveGrid extends StatelessWidget {
  const AdaptiveGrid({super.key, required this.itemCount, required this.itemBuilder});

  final int itemCount;
  final IndexedWidgetBuilder itemBuilder;

  @override
  Widget build(BuildContext context) => GridView.builder(
        gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
          maxCrossAxisExtent: 240,
          crossAxisSpacing: AppSpacing.base,
          mainAxisSpacing: AppSpacing.base,
        ),
        itemCount: itemCount,
        itemBuilder: itemBuilder,
      );
}
```

`maxCrossAxisExtent` picks the column count with no breakpoint at all.
