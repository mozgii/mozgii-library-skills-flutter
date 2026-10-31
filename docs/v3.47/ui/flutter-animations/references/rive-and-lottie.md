---
title: Flutter Rive data binding and Lottie control
author: MOZGII, Inc.
lang: en
tags: [flutter, animations, rive, lottie]
description: Driving a Rive 0.14 file from app state with Data Binding, and controlling or theming Lottie animations at runtime.
---
# Flutter Rive Data Binding and Lottie Control

Designer-made animation stays useful when the app can drive it. Rive takes live values through Data Binding, and Lottie takes a controller and color overrides.

```yaml
dependencies:
  rive: ^0.14.11
  lottie: ^3.8.1
```

## Initialize the Rive native runtime

`rive` 0.14 runs on the C++ runtime, which has to be ready before the first frame.

```dart
import 'package:material_ui/material_ui.dart';
import 'package:rive/rive.dart' as rive;

Future<void> startApp(Widget app) async {
  WidgetsFlutterBinding.ensureInitialized();
  await rive.RiveNative.init();
  runApp(app);
}
```

## Rive driven by app state through Data Binding

Bind the artboard's view model, then write its properties from a provider listener. The property names come from the Rive file. Load the file yourself here, because the widget needs the view model that `dataBind` returns. Check `mounted` after the await, and dispose all three objects.

The listener below uses a Dart `switch` on a sealed auth state. Freezed 3.0 removed `when` and `map`, and Freezed 3.1 added them back, so `next.maybeWhen(...)` also compiles under Freezed 4; a `switch` works with or without code generation.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';
import 'package:rive/rive.dart' as rive;

sealed class AuthState {
  const AuthState();
}

class AuthSignedOut extends AuthState {
  const AuthSignedOut();
}

class AuthLoading extends AuthState {
  const AuthLoading();
}

class AuthSignedIn extends AuthState {
  const AuthSignedIn();
}

class AuthFailed extends AuthState {
  const AuthFailed();
}

/// Yours: bind this to your auth notifier.
final authProvider = Provider<AuthState>((ref) => const AuthSignedOut());

class AuthButtonRive extends ConsumerStatefulWidget {
  const AuthButtonRive({super.key});

  @override
  ConsumerState<AuthButtonRive> createState() => _AuthButtonRiveState();
}

class _AuthButtonRiveState extends ConsumerState<AuthButtonRive> {
  rive.File? _file;
  rive.RiveWidgetController? _controller;
  rive.ViewModelInstance? _viewModel;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final file = await rive.File.asset(
      'assets/animations/auth_button.riv',
      riveFactory: rive.Factory.rive,
    );
    if (file == null) return;
    final controller = rive.RiveWidgetController(file);
    final viewModel = controller.dataBind(rive.DataBind.auto());
    if (!mounted) {
      viewModel.dispose();
      controller.dispose();
      file.dispose();
      return;
    }
    setState(() {
      _file = file;
      _controller = controller;
      _viewModel = viewModel;
    });
  }

  @override
  void dispose() {
    _viewModel?.dispose();
    _controller?.dispose();
    _file?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    ref.listen(authProvider, (previous, next) {
      final viewModel = _viewModel;
      if (viewModel == null) return;
      viewModel.boolean('isLoading')?.value = next is AuthLoading;
      switch (next) {
        case AuthSignedIn():
          viewModel.trigger('onSuccess')?.trigger();
        case AuthFailed():
          viewModel.trigger('onError')?.trigger();
        case AuthLoading() || AuthSignedOut():
          break;
      }
    });
    final controller = _controller;
    if (controller == null) return const SizedBox.shrink();
    return rive.RiveWidget(controller: controller);
  }
}
```

## Controlled Lottie with an AnimationController

Pass a controller to play Lottie on your own schedule. Its duration comes from the file, so set it in `onLoaded`.

```dart
import 'package:lottie/lottie.dart';
import 'package:material_ui/material_ui.dart';

class SuccessCheck extends StatefulWidget {
  const SuccessCheck({super.key});

  @override
  State<SuccessCheck> createState() => _SuccessCheckState();
}

class _SuccessCheckState extends State<SuccessCheck>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller = AnimationController(vsync: this);

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Lottie.asset(
        'assets/animations/success_check.json',
        controller: _controller,
        onLoaded: (composition) {
          _controller
            ..duration = composition.duration
            ..forward();
        },
      );
}
```

## Lottie dynamic colors with ValueDelegate

`ValueDelegate` overrides a property at runtime, with no reload of the asset, so a file can follow the app's theme. Because this widget reads `Theme.of(context)`, it rebuilds when the theme changes.

```dart
import 'package:lottie/lottie.dart';
import 'package:material_ui/material_ui.dart';

class ThemedLottie extends StatelessWidget {
  const ThemedLottie({super.key, required this.assetPath});

  final String assetPath;

  @override
  Widget build(BuildContext context) {
    final colors = Theme.of(context).colorScheme;
    return Lottie.asset(
      assetPath,
      frameRate: FrameRate.max,
      delegates: LottieDelegates(
        values: [
          ValueDelegate.color(const ['Primary Layer', '**'], value: colors.primary),
          ValueDelegate.color(const ['Accent Layer', '**'], value: colors.secondary),
          ValueDelegate.color(
            const ['Background', 'Rectangle', 'Fill 1'],
            value: colors.surfaceContainerHighest,
          ),
        ],
      ),
    );
  }
}
```

To find a layer path, open the `.json` file and look for the `"nm"` (name) keys of each layer and shape. A path must match the name hierarchy exactly, `['LayerName', 'ShapeName', 'FillOrStrokeName']`, and `'**'` stands for all children.
