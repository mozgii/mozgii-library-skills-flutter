---
title: A dark mode toggle button
author: MOZGII, Inc.
lang: en
tags: [flutter, theme, dark-mode, accessibility]
description: An animated dark and light theme toggle that works anywhere in the app and tells a screen reader what it does.
---
# A Dark Mode Toggle Button

The toggle reads `themeModeProvider` and can sit anywhere in the app. It needs a `tooltip`, because an icon on its own reads as nothing to a screen reader.

```dart
// lib/shared/widgets/theme_toggle_button.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../../core/state/theme_notifier.dart';

class ThemeToggleButton extends ConsumerWidget {
  const ThemeToggleButton({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final mode = ref.watch(themeModeProvider);
    return IconButton(
      tooltip: mode == ThemeMode.dark ? 'Use light theme' : 'Use dark theme',
      icon: AnimatedSwitcher(
        duration: const Duration(milliseconds: 300),
        transitionBuilder: (child, animation) =>
            RotationTransition(turns: animation, child: child),
        child: Icon(
          mode == ThemeMode.dark ? Icons.light_mode : Icons.dark_mode,
          key: ValueKey(mode),
        ),
      ),
      onPressed: () => ref.read(themeModeProvider.notifier).toggleTheme(),
    );
  }
}
```
