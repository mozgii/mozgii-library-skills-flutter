---
title: flutter_animate effects and patterns
author: MOZGII, Inc.
lang: en
tags: [flutter, animations, flutter_animate]
description: Staggered rows, state-driven effects, press feedback, and the blur, saturate, callback and swap effects of flutter_animate 4.5.2.
---
# flutter_animate Effects and Patterns

`flutter_animate` covers most microinteractions in a line or two. This reference holds the patterns past a basic chain. The package doesn't check Reduce Motion, so gate each one as the skill shows.

## Staggered rows and state-driven effects

A stagger delays each row by its index; clamp the index, or row 200 waits 16 seconds. `target` animates toward a value when it changes, so state can drive an effect without a controller.

```dart
import 'package:material_ui/material_ui.dart';
import 'package:flutter_animate/flutter_animate.dart';

class StaggeredRow extends StatelessWidget {
  const StaggeredRow({super.key, required this.index});

  final int index;

  @override
  Widget build(BuildContext context) => ListTile(title: Text('Item $index'))
      .animate(delay: (index.clamp(0, 8) * 80).ms)
      .fadeIn()
      .slideX(begin: -0.2, end: 0);
}

class ErrorShake extends StatelessWidget {
  const ErrorShake({super.key, required this.hasError, required this.child});

  final bool hasError;
  final Widget child;

  @override
  Widget build(BuildContext context) => child
      .animate(target: hasError ? 1 : 0)
      .shake(duration: 400.ms, hz: 4);
}
```

## Press feedback with your own controller

Pass your own controller with `.animate(controller: controller, autoPlay: false)`, then call `forward` and `reverse` from the gesture. The caller owns that controller and disposes it.

```dart
import 'package:flutter_animate/flutter_animate.dart';
import 'package:material_ui/material_ui.dart';

class PressScale extends StatefulWidget {
  const PressScale({super.key, required this.child});

  final Widget child;

  @override
  State<PressScale> createState() => _PressScaleState();
}

class _PressScaleState extends State<PressScale>
    with SingleTickerProviderStateMixin {
  late final AnimationController _press = AnimationController(vsync: this);

  @override
  void dispose() {
    _press.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => GestureDetector(
        onTapDown: (_) => _press.forward(),
        onTapUp: (_) => _press.reverse(),
        onTapCancel: () => _press.reverse(),
        child: widget.child
            .animate(controller: _press, autoPlay: false)
            .scale(
              begin: const Offset(1, 1),
              end: const Offset(0.95, 0.95),
              duration: 100.ms,
            ),
      );
}
```

## Blur, saturate, callback and swap effects

`.blur` takes two `Offset`s for x and y sigma. `.desaturate` runs from 1 to 0 by default, and `.saturate` the other way. `.callback` fires at a point in the chain and passes a `bool` that says whether it's playing in reverse.

`.swap` replaces the widget it's chained on, including earlier effects, when the chain reaches it. So start from the old widget and return the new one from `builder`:

```dart
import 'package:flutter_animate/flutter_animate.dart';
import 'package:material_ui/material_ui.dart';

class LoadingCrossfade extends StatelessWidget {
  const LoadingCrossfade({super.key, required this.placeholder, required this.content});

  final Widget placeholder;
  final Widget content;

  @override
  Widget build(BuildContext context) => placeholder
      .animate()
      .fadeOut(duration: 300.ms)
      .swap(builder: (context, _) => content.animate().fadeIn(duration: 300.ms));
}
```
