---
name: flutter-add-widget-preview
description: Adds Flutter Widget Previewer previews with the @Preview annotation so a widget renders live and alone, in several themes, sizes and text scales - custom annotations, MultiPreview, PreviewThemeData, runtime transform. Use when asked to preview a widget, see a component in dark mode, build a design catalogue without running the app, or fix a preview that throws or renders blank.
metadata:
  tags:
  - flutter
  - ui
  - previews
---
# Flutter Widget Previews

The Widget Previewer renders a widget by itself, outside the app, and updates it as you type. You can check a card in light and dark, at 200% text, or at a phone's width without navigating to it. It's stable as of Flutter 3.47.

## Step 0: Check the SDK

```bash
flutter --version
flutter widget-preview --help
```

Previews live in your normal source files, and the annotation comes from the SDK, so no dependency is needed. `.widget_preview/` holds the build cache: keep it out of git.

## Choosing a Preview Style

| You want | Use | Why |
|---|---|---|
| One widget, one look | `@Preview(name: ...)` on a top-level function | The least code |
| Two or three fixed variants | Several `@Preview` annotations on one function | No new class |
| The same setup on many widgets | A subclass of `Preview` | The theme and wrapper live in one place |
| A matrix such as light and dark, repeated | A subclass of `MultiPreview` | One annotation expands into many previews |
| A name or theme worked out from other values | Override `transform()` | A `const` constructor can't compute it |

## Where the Annotation Can Go

`@Preview` works on a top-level function, on a static method, and on a public widget constructor or factory with no required arguments. The target has to return a `Widget` or a `WidgetBuilder`. Import `package:flutter/widget_previews.dart`.

## Preview Parameters

| Parameter | Sets |
|---|---|
| `name` | The title on the preview card |
| `group` | Which group the card sits in; filtering matches it too |
| `size` | Artificial constraints, as a `Size` |
| `textScaleFactor` | The font scale |
| `wrapper` | A function wrapping the widget, for instance in an `InheritedWidget` that injects state |
| `theme` | A function returning a `PreviewThemeData` instance |
| `brightness` | The starting `Brightness` |
| `localizations` | A function returning the localization setup |

## Worked Example: An Expense Card in Every State

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// lib/expenses/expense_card.dart
import 'package:flutter/widget_previews.dart';
import 'package:material_ui/material_ui.dart';

class ExpenseCard extends StatelessWidget {
  const ExpenseCard({super.key, required this.note, required this.amount});

  final String note;
  final double amount;

  @override
  Widget build(BuildContext context) => Card(
        child: ListTile(
          title: Text(note),
          trailing: Text(amount.toStringAsFixed(2)),
        ),
      );
}

/// Material needs a Material ancestor, so every preview goes through this.
Widget surface(Widget child) => Material(child: Center(child: child));

/// Applies the app's seed colour in the brightness the preview asks for.
final class AppPreviewTheme extends PreviewThemeData {
  const AppPreviewTheme([this.brightness]);

  final Brightness? brightness;

  @override
  Widget apply(BuildContext context, Widget child) => Theme(
        data: ThemeData.from(
          colorScheme: ColorScheme.fromSeed(
            seedColor: Colors.indigo,
            brightness: brightness ?? Brightness.light,
          ),
        ),
        child: child,
      );
}

PreviewThemeData darkAppTheme() => const AppPreviewTheme(Brightness.dark);
PreviewThemeData lightAppTheme() => const AppPreviewTheme(Brightness.light);

/// One annotation, three cards: light, dark and large text.
final class ExpenseStates extends MultiPreview {
  const ExpenseStates(this.label);

  final String label;

  @override
  List<Preview> get previews => const [
        Preview(theme: lightAppTheme, wrapper: surface, size: Size(360, 96)),
        Preview(theme: darkAppTheme, wrapper: surface, size: Size(360, 96)),
        Preview(theme: lightAppTheme, wrapper: surface, size: Size(360, 160), textScaleFactor: 2),
      ];

  @override
  List<Preview> transform() {
    var index = 0;
    const variants = ['light', 'dark', 'large text'];
    return super.transform().map((preview) {
      final builder = preview.toBuilder()
        ..group = 'Expenses'
        ..name = '$label: ${variants[index++]}';
      return builder.build();
    }).toList();
  }
}

@ExpenseStates('Short note')
Widget shortNote() => const ExpenseCard(note: 'Coffee', amount: 3.5);

@ExpenseStates('Long note')
Widget longNote() => const ExpenseCard(
      note: 'Quarterly subscription for the shared design tool',
      amount: 1240,
    );
```

`PreviewBuilder` has `build()`; a preview's `toBuilder()` gives you one, and `transform()` is the place to use it. `theme`, `wrapper` and `localizations` must be public functions, because the previewer's generated code calls them by name.

## Running the Previewer

Android Studio, IntelliJ and VS Code start the previewer when the project opens. The previews sit in the sidebar tab named "Flutter Widget Preview"; switch off "Filter previews by selected file" to see previews outside the active file. From a terminal:

```bash
flutter widget-preview start
```

This serves the previewer locally and opens it in Chrome. Edits show up on their own. After changing static state, use the restart control for the whole previewer, in its lower right corner; to reset one card's local state, use the restart control on that card. An IDE session handles one project or Pub workspace at a time.

## What the Previewer Can't Run

The previewer is a web app, and that sets its limits:

- Native plugins, `dart:io` and `dart:ffi` aren't available. A widget with such a dependency loads, then throws when the API is called. Put the call behind an interface, or use a conditional import to swap in a stub.
- Assets loaded with `fromAsset` APIs need a package path: `packages/expense_tracker/assets/receipt.png`, not `assets/receipt.png`.
- An unconstrained widget is capped at about half the previewer's height and width, so pass `size` for anything that expands.

## Verify

```bash
flutter analyze lib/expenses/expense_card.dart
flutter widget-preview start
```

Check each card by eye: both brightnesses, the longest realistic text, and 200% text scale. Then read the console for exceptions, because a card can look right while a callback throws.

## Gotchas

1. **The preview shows an exception about `dart:io`.** Symptom: a red card naming `Platform` or a plugin. Cause: a transitive dependency reached native code. Fix: inject the dependency, and hand the preview a fake.
2. **`No Material widget found`.** Symptom: a preview of a `ListTile` or a `TextField` fails. Cause: nothing above it provides `Material`. Fix: add a `wrapper` such as the `surface` function above.
3. **The image is blank.** Symptom: an empty box where the picture belongs. Cause: a plain asset path. Fix: use the `packages/<name>/` path.
4. **A const expression error on `theme:`.** Symptom: the analyzer rejects a lambda in the annotation. Cause: annotation arguments must be constant. Fix: pass a public top-level function by name.
5. **The widget fills half the window.** Symptom: an odd size for an expanding widget. Cause: the default cap on unconstrained widgets. Fix: set `size`.
6. **Static state stays stale.** Symptom: a change to a top-level value doesn't show. Cause: hot reload keeps statics. Fix: use the restart control for the whole previewer.

## Anti-Patterns

- Previewing a whole screen that needs a router and a backend. Preview its parts, and inject the data.
- Copying theme code into every preview. Subclass `Preview` or `MultiPreview`.
- Previewing only the happy state. Add the empty, error and long-text cases.

## Pre-Merge Checklist

- [ ] The widget previews in light and dark
- [ ] One preview uses a large `textScaleFactor`
- [ ] Previews use `name` and `group`, so they're easy to find
- [ ] No preview needs a plugin or `dart:io`
- [ ] `.widget_preview/` is in `.gitignore`

## See Also

- `flutter-design-ui` for the components being previewed
- `flutter-theme` for the theme a preview should apply
- `flutter-add-widget-test` to lock in what a preview shows
