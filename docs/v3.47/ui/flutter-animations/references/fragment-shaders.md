---
title: Flutter GLSL fragment shaders
author: MOZGII, Inc.
lang: en
tags: [flutter, animations, shaders, glsl]
description: Loading a GLSL fragment shader, feeding it uniforms, repainting it without rebuilds, and shutting it down correctly.
---
# Flutter GLSL Fragment Shaders

Fragment shaders draw effects that widgets can't: particles, fluid backgrounds, per-pixel gradients. Declare each file in `pubspec.yaml` so the build compiles it for the active renderer.

```yaml
flutter:
  shaders:
    - assets/shaders/particles.frag
```

## A minimal GLSL shader for Flutter

`flutter/runtime_effect.glsl` provides `FlutterFragCoord`, which gives pixel coordinates on every backend. Uniforms are set by position, and a `vec2` takes two float slots.

```glsl
#include <flutter/runtime_effect.glsl>

uniform float uTime;
uniform vec2 uResolution;
out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / uResolution;
  float pulse = sin(uTime * 2.0 + uv.x * 10.0) * 0.5 + 0.5;
  fragColor = vec4(0.42, 0.31, 1.0, pulse * 0.6);
}
```

## ShaderBackground widget with a clean lifecycle

Pass the controller as the painter's `repaint`, so a new frame repaints the canvas without rebuilding any widget. Loading is async, so check `mounted` before `setState`, and dispose both the controller and the shader. The time controller stops when motion is off, which leaves a still frame.

```dart
import 'dart:ui' as ui;

import 'package:material_ui/material_ui.dart';

class ShaderBackground extends StatefulWidget {
  const ShaderBackground({super.key});

  @override
  State<ShaderBackground> createState() => _ShaderBackgroundState();
}

class _ShaderBackgroundState extends State<ShaderBackground>
    with SingleTickerProviderStateMixin {
  static const _seconds = 60;

  ui.FragmentShader? _shader;
  late final AnimationController _time = AnimationController(
    vsync: this,
    duration: const Duration(seconds: _seconds),
  );

  @override
  void initState() {
    super.initState();
    _load();
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (MediaQuery.disableAnimationsOf(context)) {
      _time.stop();
    } else if (!_time.isAnimating) {
      _time.repeat();
    }
  }

  Future<void> _load() async {
    final program =
        await ui.FragmentProgram.fromAsset('assets/shaders/particles.frag');
    final shader = program.fragmentShader();
    if (!mounted) {
      shader.dispose();
      return;
    }
    setState(() => _shader = shader);
  }

  @override
  void dispose() {
    _time.dispose();
    _shader?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final shader = _shader;
    if (shader == null) return const SizedBox.shrink();
    return RepaintBoundary(
      child: CustomPaint(
        painter: ShaderPainter(shader: shader, time: _time, seconds: _seconds),
        size: Size.infinite,
      ),
    );
  }
}

class ShaderPainter extends CustomPainter {
  ShaderPainter({required this.shader, required this.time, required this.seconds})
      : super(repaint: time);

  final ui.FragmentShader shader;
  final Animation<double> time;
  final int seconds;

  @override
  void paint(Canvas canvas, Size size) {
    shader
      ..setFloat(0, time.value * seconds)
      ..setFloat(1, size.width)
      ..setFloat(2, size.height);
    canvas.drawRect(Offset.zero & size, Paint()..shader = shader);
  }

  @override
  bool shouldRepaint(ShaderPainter oldDelegate) => oldDelegate.shader != shader;
}
```
