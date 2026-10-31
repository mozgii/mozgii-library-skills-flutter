---
name: flutter-animations
description: Builds Flutter motion that holds its frame rate and respects Reduce Motion - flutter_animate chains, AnimationController lifecycles, physics springs, Rive 0.14 with RiveWidgetBuilder and data binding, Lottie, Hero, go_router page transitions, predictive back, Material Motion and GLSL fragment shaders. Use when asked to animate a screen or widget, add a microinteraction, play a Rive or After Effects file, build a custom page transition, make a card spring back, or fix an animation that leaks, stutters or ignores accessibility settings.
metadata:
  tags:
  - flutter
  - animations
  - rive
  - lottie
  - motion
  - shaders
---
# Flutter Animations

Motion tells people what changed and where things went. This skill picks the cheapest tool that does the job, builds it with a correct lifecycle, and makes sure it stops for anyone who asked for less motion.

Three rules hold throughout: every controller is disposed, every animation has a Reduce Motion path, and nothing misses its frame on a mid-range Android phone.

## Step 0: Read the Project's Versions

These examples were checked against Flutter 3.47.9, `material_ui` 1.9.0, `flutter_animate` 4.5.2, `rive` 0.14.11, `lottie` 3.8.1, `animations` 3.0.0 and `go_router` 18.1.2.

```bash
flutter --version
grep -nE "^\s+(material_ui|flutter_animate|rive|lottie|animations|go_router):" pubspec.yaml
```

Rive below 0.14 has a different API, so check it first. A project still on `package:flutter/material.dart` moves with `dart fix --apply --code=migrate_design_widgets`. The examples also import `cupertino_ui: ^1.1.5`, `flutter_animate: ^4.5.2`, `flutter_riverpod: ^3.4.3`, `go_router: ^18.1.2`, `material_ui: ^1.9.0`.

## Which Animation Tool to Use

| You need | Use | Why the others lose here |
|---|---|---|
| One property eases to a new value | `AnimatedContainer`, `AnimatedOpacity` | A controller adds lifecycle code for nothing |
| An entrance, a press or a chain of effects | `flutter_animate` | Hand-built controllers take 40 lines for the same fade and slide |
| Exact timing, or several values in step | `AnimationController` | `flutter_animate` hides the controller you need to drive |
| Motion that continues from a finger's velocity | `SpringSimulation` | A fixed curve ignores the speed of the gesture |
| A designer's interactive character or state machine | Rive | Lottie plays a timeline and takes no live input |
| A designer's After Effects loop | Lottie | Rewriting it in code loses the designer's control |
| Moving between routes | `Hero`, `CustomTransitionPage`, `animations` | Overlays by hand break the back gesture |
| Per-pixel effects | Fragment shader | Widgets can't reach individual pixels |

## Reduce Motion: Read Both Platform Flags

`MediaQuery.disableAnimationsOf` covers Android's "Remove animations" setting. It does not cover iOS: there, Reduce Motion sets only `accessibilityFeatures.reduceMotion`, which `MediaQuery` never reads. Merge the two once at the top of the app, and every widget below can ask one question.

```dart
// lib/motion/motion_aware_app.dart
import 'package:material_ui/material_ui.dart';

class MotionPreference extends StatefulWidget {
  const MotionPreference({super.key, required this.child});

  final Widget child;

  @override
  State<MotionPreference> createState() => _MotionPreferenceState();
}

class _MotionPreferenceState extends State<MotionPreference>
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAccessibilityFeatures() => setState(() {});

  @override
  Widget build(BuildContext context) {
    final data = MediaQuery.of(context);
    final features =
        WidgetsBinding.instance.platformDispatcher.accessibilityFeatures;
    return MediaQuery(
      data: data.copyWith(
        disableAnimations: data.disableAnimations || features.reduceMotion,
      ),
      child: widget.child,
    );
  }
}

class MotionAwareApp extends StatelessWidget {
  const MotionAwareApp({super.key, required this.home});

  final Widget home;

  @override
  Widget build(BuildContext context) => MaterialApp(
        builder: (context, child) => MotionPreference(child: child!),
        home: home,
      );
}
```

Below that scope, `MediaQuery.disableAnimationsOf(context)` is the one question. What to do with the answer depends on the tool:

| Tool | When motion is off |
|---|---|
| `flutter_animate` | Return the plain child; the package doesn't check this setting |
| `AnimationController` | Set `value` directly, and don't call `repeat` |
| Lottie | `animate: false` |
| Shimmer | `enabled: false` |
| Custom page transition | `transitionDuration: Duration.zero` |
| Hero | Wrap in `HeroMode(enabled: false, child: ...)` |
| Shader loop | Stop the time controller |

Progress indicators can keep moving, since they carry information. Parallax, loops and large slides should go.

## flutter_animate Chains

`flutter_animate` adds `.animate()` to any widget and chains effects. Effects in a chain start together unless `.then()` moves the start point on. The gate at the top is the Reduce Motion path.

```dart
import 'package:flutter_animate/flutter_animate.dart';
import 'package:material_ui/material_ui.dart';

class WelcomeTitle extends StatelessWidget {
  const WelcomeTitle({super.key});

  @override
  Widget build(BuildContext context) {
    const title = Text('Welcome back');
    if (MediaQuery.disableAnimationsOf(context)) return title;
    return title
        .animate()
        .fadeIn(duration: 600.ms, curve: Curves.easeOut)
        .slideY(begin: 0.3, end: 0, duration: 600.ms, curve: Curves.easeOut)
        .then(delay: 200.ms)
        .shimmer(duration: 1200.ms);
  }
}
```

Staggered rows, state-driven effects, press feedback, and the `blur`, `saturate`, `callback` and `swap` effects are in [references/flutter-animate-effects.md](references/flutter-animate-effects.md). There is no tilt effect in 4.5.2: `.flip` or `.rotate` cover that look.

## Explicit AnimationController Lifecycle

Reach for a controller when timing is exact or several values move together. It needs a `TickerProvider`, and `dispose` must run: a `State` that leaves one running fails with `was disposed with an active Ticker`.

```dart
// lib/motion/pulsing_dot.dart
import 'package:material_ui/material_ui.dart';

class PulsingDot extends StatefulWidget {
  const PulsingDot({super.key});

  @override
  State<PulsingDot> createState() => _PulsingDotState();
}

class _PulsingDotState extends State<PulsingDot>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller = AnimationController(
    duration: const Duration(milliseconds: 1200),
    vsync: this,
  );
  late final CurvedAnimation _curve =
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut);
  late final Animation<double> _scale =
      Tween<double>(begin: 0.8, end: 1.2).animate(_curve);

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (MediaQuery.disableAnimationsOf(context)) {
      _controller.value = 0.5; // a still, mid-size dot
    } else if (!_controller.isAnimating) {
      _controller.repeat(reverse: true);
    }
  }

  @override
  void dispose() {
    _curve.dispose();
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => ScaleTransition(
        scale: _scale,
        child: const CircleAvatar(radius: 24),
      );
}
```

`ScaleTransition`, `FadeTransition` and `SlideTransition` rebuild only themselves. Use them before `AnimatedBuilder`, and when you do use `AnimatedBuilder`, pass static content as `child`.

## Physics-Based Spring Animations

Spring Animations start from where the finger left off, at the finger's speed. This worked example drags a card and lets it spring home. Drive an unbounded controller in the same units as the drag, so the velocity needs no conversion: pixels and pixels per second.

`SpringCard`, the whole widget, is in `references/transitions.md`.

## Rive 0.14 with RiveWidgetBuilder

Rive is the tool for interactive vector characters and UI state machines. `rive` 0.14 replaced the Dart runtime with the C++ runtime and removed `RiveAnimation`, `Rive(artboard:)`, `RiveFile.import`, `StateMachineController` and the `SMI*` input classes. If Rive code stopped compiling, this is why. Call `await RiveNative.init()` in `main` before `runApp`, then let `RiveWidgetBuilder` own the file and controller and report loading, loaded and failed states.

```dart
import 'package:material_ui/material_ui.dart';
import 'package:rive/rive.dart' as rive;

class RiveSpinner extends StatefulWidget {
  const RiveSpinner({super.key});

  @override
  State<RiveSpinner> createState() => _RiveSpinnerState();
}

class _RiveSpinnerState extends State<RiveSpinner> {
  final _loader = rive.FileLoader.fromAsset(
    'assets/animations/loading_spinner.riv',
    riveFactory: rive.Factory.rive,
  );

  @override
  void dispose() {
    _loader.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => SizedBox.square(
        dimension: 64,
        child: rive.RiveWidgetBuilder(
          fileLoader: _loader,
          builder: (context, state) => switch (state) {
            rive.RiveLoading() => const SizedBox.shrink(),
            rive.RiveFailed() => const Icon(Icons.error_outline),
            rive.RiveLoaded(:final controller) =>
              rive.RiveWidget(controller: controller),
          },
        ),
      );
}
```

Import it with a prefix: `rive` exports its own `File` and `Image`, which collide with `dart:io` and widgets. Rive now recommends Data Binding over state machine inputs, which still work but are deprecated; driving a file from app state is in [references/rive-and-lottie.md](references/rive-and-lottie.md).

## Lottie for After Effects Files

Lottie plays an After Effects animation exported as JSON. Use `Lottie.asset`, and pass `frameRate: FrameRate.max` so a 24 fps file renders smoothly on a fast display.

```dart
import 'package:lottie/lottie.dart';
import 'package:material_ui/material_ui.dart';

class CelebrationBurst extends StatelessWidget {
  const CelebrationBurst({super.key});

  @override
  Widget build(BuildContext context) => Lottie.asset(
        'assets/animations/celebration.json',
        width: 200,
        height: 200,
        fit: BoxFit.contain,
        frameRate: FrameRate.max,
        repeat: false,
        animate: !MediaQuery.disableAnimationsOf(context),
      );
}
```

Driving Lottie from a controller, and theming layers with `ValueDelegate`, are in [references/rive-and-lottie.md](references/rive-and-lottie.md).

## Hero and Route Transitions

`Hero` flies a widget between two routes that share a `tag`. Tags must be unique per route, so build them from the type and id, such as `'product-image-${product.id}'`. Keep the child the same widget type on both sides, and use `flightShuttleBuilder` when the two shapes differ.

Custom go_router transitions, predictive back, `SharedAxisTransition`, `FadeThroughTransition`, `OpenContainer` and a navigation bar that hides on scroll are in [references/transitions.md](references/transitions.md). Fragment shaders are in [references/fragment-shaders.md](references/fragment-shaders.md).

## 120 Hz Displays and Frame Rate

A `Ticker` fires once per frame the engine produces, so `AnimationController` and `flutter_animate` follow the display's rate on their own. iPhones with ProMotion need `CADisableMinimumFrameDurationOnPhone` set to `true` in `Info.plist`: `flutter create` adds it, but a project created long ago may not have it. On Android the system picks the display mode, so check `display.refreshRate` on the device rather than assuming 120.

Lottie is the exception. Its default holds to the file's own frames, so pass `FrameRate.max`. Use `FrameRate.composition` only when the frames must stay exact, such as sync with audio.

## Verify

The examples were compiled, not run on a device, so judge motion and frame rate on real hardware. Run `flutter analyze`, `flutter test` and `flutter run --profile`. Check these by hand on a real device: turn on Reduce Motion on iOS and "Remove animations" on Android, and confirm nothing loops or slides. Then record the animation in DevTools' Performance tab and confirm no frame runs over budget. A widget test pins the gate:

```dart
import 'package:app/motion/motion_aware_app.dart';
import 'package:app/motion/pulsing_dot.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('iOS Reduce Motion stops the pulsing dot', (tester) async {
    tester.platformDispatcher.accessibilityFeaturesTestValue =
        const FakeAccessibilityFeatures(reduceMotion: true);
    addTearDown(tester.platformDispatcher.clearAccessibilityFeaturesTestValue);

    await tester.pumpWidget(const MotionAwareApp(home: PulsingDot()));
    await tester.pump();

    expect(tester.binding.hasScheduledFrame, isFalse);
  });
}
```

`FakeAccessibilityFeatures` also takes `disableAnimations: true`, which stands in for Android's setting.

## Gotchas

1. **`AnimationController.forward() called after AnimationController.dispose()`.** Symptom: a red screen after leaving a page. Cause: a timer or future used the controller after `dispose`. Fix: cancel the timer in `dispose`, and check `mounted` after every `await`.
2. **`was disposed with an active Ticker`.** Symptom: a framework error when a page closes. Cause: no `dispose` in the `State`, or a controller still running. Fix: dispose every controller and `CurvedAnimation`.
3. **Reduce Motion on, animations still running on iPhone.** Symptom: an accessibility report on iOS only. Cause: `disableAnimations` stays false there. Fix: merge `accessibilityFeatures.reduceMotion` as in `MotionPreference`.
4. **`The method 'tilt' isn't defined`.** Symptom: an analyzer error from an example found online. Cause: `flutter_animate` has no such effect. Fix: use `.flip` or `.rotate`.
5. **`.swap` shows the placeholder forever.** Symptom: real content never appears. Cause: the chain was started on the content and the builder returned the placeholder. Fix: chain on the old widget and build the new one.
6. **Spring flies off screen.** Symptom: the card overshoots wildly on release. Cause: the velocity was scaled to a different unit from the controller's value. Fix: use pixels for both.
7. **Rows re-animate while scrolling.** Symptom: list items fade in again on every scroll back. Cause: `ListView.builder` rebuilt the row, so `.animate()` replayed. Fix: animate only on first load, or give the row's state a "seen" flag.
8. **Lottie looks choppy at 120 Hz.** Symptom: visible steps in a smooth curve. Cause: the default keeps the file's frame rate. Fix: `frameRate: FrameRate.max`.
9. **`The method 'when' isn't defined` on a Freezed class.** Symptom: a state switch fails after an upgrade. Cause: Freezed 3.0 removed `when` and `map`. Fix: use Freezed 3.1 or later, which added them back, or a Dart `switch`.

## Anti-patterns

| Don't | Do instead |
|---|---|
| Rebuild a whole page from `AnimatedBuilder` | Pass static content as `child`, or use a `*Transition` widget |
| Use `setState` in a controller listener | Use `AnimatedBuilder` or a transition widget |
| Leave a `repeat()` running off screen | Stop it, or let `TickerMode` silence it |
| Animate layout-heavy properties every frame | Animate `Transform` and opacity, which don't relayout |

## Pre-merge Checklist

- [ ] Every controller, `CurvedAnimation` and shader is disposed
- [ ] No `setState` after an `await` without a `mounted` check
- [ ] Reduce Motion tested on iOS and Android, with no loops or large slides
- [ ] Static content passed as `child`
- [ ] No frame over budget in profile mode on a mid-range phone
- [ ] Rive and Lottie files show something sensible on failure

## See Also

- `flutter-performance` for profiling and frame budgets
- `flutter-setup-declarative-routing` for go_router itself
- `flutter-design-ui` for shimmer loading and the components being animated
