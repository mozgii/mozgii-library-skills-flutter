---
title: Flutter route transitions and Material Motion
author: MOZGII, Inc.
lang: en
tags: [flutter, animations, go_router, material-motion]
description: Custom go_router page transitions, predictive back on Android, the animations package patterns, and a navigation bar that hides on scroll.
---
# Flutter Route Transitions and Material Motion

Route transitions tell people how screens relate. Keep the platform's default unless a design needs more, and make any custom one honor Reduce Motion.

## Custom page transition with go_router

`CustomTransitionPage` sets the transition per route. Reading `MediaQuery` needs a `BuildContext`, which `pageBuilder` gives you, so the duration can drop to zero for people who asked for less motion. `CupertinoPage` keeps the iOS swipe-back gesture; a `CustomTransitionPage` doesn't have it.

```dart
import 'package:material_ui/material_ui.dart';
import 'package:cupertino_ui/cupertino_ui.dart';
import 'package:go_router/go_router.dart';

GoRoute productRoute(Widget Function(String id) page) => GoRoute(
      path: '/product/:id',
      pageBuilder: (context, state) => CustomTransitionPage<void>(
        key: state.pageKey,
        child: page(state.pathParameters['id']!),
        transitionDuration: MediaQuery.disableAnimationsOf(context)
            ? Duration.zero
            : const Duration(milliseconds: 350),
        transitionsBuilder: (context, animation, secondaryAnimation, child) =>
            SlideTransition(
          position: animation.drive(
            Tween(begin: const Offset(1, 0), end: Offset.zero)
                .chain(CurveTween(curve: Curves.easeOutCubic)),
          ),
          child: child,
        ),
      ),
    );

GoRoute settingsRoute(Widget page) => GoRoute(
      path: '/settings',
      pageBuilder: (context, state) => CupertinoPage<void>(
        key: state.pageKey,
        child: page,
      ),
    );
```

## Predictive back gesture on Android

Predictive back shows the destination peeking behind the current screen before the person commits to going back. Since Flutter 3.38 the predictive builder is Android's default, so set it only when a theme has overridden it.

```dart
import 'package:cupertino_ui/cupertino_ui.dart';
import 'package:material_ui/material_ui.dart';

final predictiveTheme = ThemeData(
  pageTransitionsTheme: const PageTransitionsTheme(
    builders: {
      TargetPlatform.android: PredictiveBackPageTransitionsBuilder(),
      TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
    },
  ),
);
```

It works with `GoRouter` with no route changes. Android's current guide calls predictive back on by default and describes `android:enableOnBackInvokedCallback="false"` as the opt-out. If the gesture doesn't appear, check that flag first; on an older target SDK, set it to `true` on the `<application>` tag:

```xml
<application
    android:enableOnBackInvokedCallback="true"
    ...>
```

## Material Motion with the animations package

The `animations` package ships the Material transition patterns.

```yaml
dependencies:
  animations: ^3.0.0
```

### SharedAxisTransition for related steps

`SharedAxisTransition` slides and fades along one axis. Use `horizontal` for a flow of pages, `vertical` for steps stacked on top of each other, and `scaled` for moving in and out of a hierarchy. `PageTransitionSwitcher` runs it, or the next pattern, whenever its child changes key.

```dart
import 'package:animations/animations.dart';
import 'package:material_ui/material_ui.dart';

class WizardBody extends StatelessWidget {
  const WizardBody({super.key, required this.step, required this.child});

  final int step;
  final Widget child;

  @override
  Widget build(BuildContext context) => PageTransitionSwitcher(
        transitionBuilder: (child, animation, secondaryAnimation) =>
            SharedAxisTransition(
          animation: animation,
          secondaryAnimation: secondaryAnimation,
          transitionType: SharedAxisTransitionType.horizontal,
          child: child,
        ),
        child: KeyedSubtree(key: ValueKey(step), child: child),
      );
}
```

### FadeThroughTransition for tab switches

`FadeThroughTransition` suits siblings with no spatial link, such as bottom navigation tabs. The old page fades out, then the new one fades in while scaling up slightly.

```dart
import 'package:animations/animations.dart';
import 'package:material_ui/material_ui.dart';

class TabBody extends StatelessWidget {
  const TabBody({super.key, required this.index, required this.pages});

  final int index;
  final List<Widget> pages;

  @override
  Widget build(BuildContext context) => PageTransitionSwitcher(
        transitionBuilder: (child, animation, secondaryAnimation) =>
            FadeThroughTransition(
          animation: animation,
          secondaryAnimation: secondaryAnimation,
          child: child,
        ),
        child: KeyedSubtree(key: ValueKey(index), child: pages[index]),
      );
}
```

### OpenContainer for a card that expands

`OpenContainer` is the container transform: a card grows into the destination screen.

```dart
import 'package:animations/animations.dart';
import 'package:material_ui/material_ui.dart';

class ProductTile extends StatelessWidget {
  const ProductTile({super.key, required this.detail});

  final Widget detail;

  @override
  Widget build(BuildContext context) => OpenContainer<void>(
        transitionDuration: const Duration(milliseconds: 400),
        transitionType: ContainerTransitionType.fade,
        openBuilder: (context, close) => detail,
        closedBuilder: (context, open) => InkWell(
          onTap: open,
          child: const ListTile(title: Text('Product')),
        ),
        closedShape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
        ),
        closedColor: Theme.of(context).colorScheme.surface,
      );
}
```

## Navigation bar that hides on scroll

This shell hides the bottom bar on scroll down and brings it back on scroll up. A `ValueNotifier` keeps the rebuild to the bar alone, and it needs its own `dispose`. Pass the same `ScrollController` to the `CustomScrollView`; a `SliverAppBar` collapses independently of this.

```dart
import 'package:material_ui/material_ui.dart';

class ScrollAwareNavBar extends StatefulWidget {
  const ScrollAwareNavBar({
    super.key,
    required this.body,
    required this.navBar,
    required this.scrollController,
  });

  final Widget body;
  final Widget navBar;
  final ScrollController scrollController;

  @override
  State<ScrollAwareNavBar> createState() => _ScrollAwareNavBarState();
}

class _ScrollAwareNavBarState extends State<ScrollAwareNavBar> {
  final _visible = ValueNotifier<bool>(true);
  double _lastOffset = 0;

  @override
  void initState() {
    super.initState();
    widget.scrollController.addListener(_onScroll);
  }

  @override
  void didUpdateWidget(ScrollAwareNavBar oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.scrollController != widget.scrollController) {
      oldWidget.scrollController.removeListener(_onScroll);
      widget.scrollController.addListener(_onScroll);
    }
  }

  void _onScroll() {
    final offset = widget.scrollController.offset;
    final delta = offset - _lastOffset;
    _lastOffset = offset;
    if (delta > 4 && _visible.value) {
      _visible.value = false;
    } else if (delta < -4 && !_visible.value) {
      _visible.value = true;
    }
  }

  @override
  void dispose() {
    widget.scrollController.removeListener(_onScroll);
    _visible.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        body: widget.body,
        bottomNavigationBar: ValueListenableBuilder<bool>(
          valueListenable: _visible,
          builder: (context, visible, child) => AnimatedSize(
            duration: const Duration(milliseconds: 250),
            curve: Curves.easeInOut,
            child: visible ? child : const SizedBox.shrink(),
          ),
          child: widget.navBar,
        ),
      );
}
```

## SpringCard: A Card That Springs Home From a Drag

`SpringCard` follows the finger, then hands the release velocity to a `SpringSimulation` on an unbounded controller, in the same units as the drag, so the card springs home at the speed it was let go.

```dart
import 'package:flutter/physics.dart';
import 'package:material_ui/material_ui.dart';

class SpringCard extends StatefulWidget {
  const SpringCard({super.key, required this.child});

  final Widget child;

  @override
  State<SpringCard> createState() => _SpringCardState();
}

class _SpringCardState extends State<SpringCard>
    with SingleTickerProviderStateMixin {
  // A ratio of 0.7 is slightly bouncy; 1.0 is critically damped.
  static final _spring = SpringDescription.withDampingRatio(
    mass: 1,
    stiffness: 500,
    ratio: 0.7,
  );

  late final AnimationController _offset =
      AnimationController.unbounded(vsync: this);

  void _onDragEnd(DragEndDetails details) {
    if (MediaQuery.disableAnimationsOf(context)) {
      _offset.value = 0;
      return;
    }
    _offset.animateWith(
      SpringSimulation(
        _spring,
        _offset.value,
        0,
        details.velocity.pixelsPerSecond.dy,
      ),
    );
  }

  @override
  void dispose() {
    _offset.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => GestureDetector(
        onVerticalDragStart: (_) => _offset.stop(),
        onVerticalDragUpdate: (details) => _offset.value += details.delta.dy,
        onVerticalDragEnd: _onDragEnd,
        child: AnimatedBuilder(
          animation: _offset,
          child: widget.child,
          builder: (context, child) => Transform.translate(
            offset: Offset(0, _offset.value),
            child: child,
          ),
        ),
      );
}
```
