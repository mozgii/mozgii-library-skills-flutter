---
name: flutter-build-responsive-layout
description: Builds Flutter layouts that adapt to any window - phones, foldables, tablets, desktop and multi-window - with MediaQuery.sizeOf, LayoutBuilder, Expanded and Flexible, breakpoints, navigation that changes form, and width caps for readable content. Use when asked to make a screen responsive, support tablets or desktop, show a grid on wide screens, fix a stretched layout, or stop the app letterboxing on foldables.
metadata:
  tags:
  - flutter
  - ui
  - responsive
---
# Flutter Responsive Layouts

A Flutter app runs in a window, and that window can be a phone, half a foldable, a floating panel or a desktop. The size of the window, rather than the kind of device, should decide the layout. Get this right and one codebase covers every form factor without branching on the platform.

## Step 0: See How the App Handles Size Today

```bash
flutter --version
grep -rnE "orientationOf|OrientationBuilder|setPreferredOrientations|isTablet|Platform\.is(Android|IOS)" lib
```

Each hit is a candidate for replacement. A layout decision that reads the device model or the orientation will misfire in split-screen. The examples were compiled, not run on a device, so resize them on a real tablet or foldable as well.

## Measuring Space: Pick the Right Tool

| Question | Use | Why the others lose here |
|---|---|---|
| How big is the app window? | `MediaQuery.sizeOf(context)` | `MediaQuery.of` rebuilds on every metric change |
| How much room did my parent give me? | `LayoutBuilder` | `MediaQuery` reports the window, not this slot |
| How big is the physical screen? | `View.of(context).display.size` | Only for apps forced into a fixed orientation |
| Should this be a list or a grid? | `SliverGridDelegateWithMaxCrossAxisExtent` | Manual column maths breaks between breakpoints |

Don't use `MediaQuery.orientationOf` or `OrientationBuilder` near the top of the tree to pick a layout. A tablet in split-screen can be portrait and narrow or landscape and tight, so orientation says nothing about the space you have. Don't check for "phone" or "tablet" either.

## How Parents and Children Agree on Size

A parent hands its child a minimum and a maximum size. The child chooses its size inside those limits and tells the parent, which then decides where it sits. Every responsive widget works inside that exchange.

## Sharing Space Inside a Row or Column

`Expanded` makes a child fill the remaining space. It's the same as `Flexible` with `fit: FlexFit.tight` and a `flex` of 1. `Flexible` lets a child be smaller than its share. The `flex` factor sets the ratio between siblings, so `flex: 2` beside `flex: 1` splits the space two to one.

## Breakpoints That Match Material

Material's window size classes give you a shared vocabulary. Use the ranges below and write them down once.

| Class | Width (dp) | Typical layout |
|---|---|---|
| Compact | under 600 | One pane, bottom navigation |
| Medium | 600 to 839 | One or two panes, navigation rail |
| Expanded | 840 to 1199 | Two panes, navigation rail |
| Large | 1200 to 1599 | Two panes, wide rail or drawer |
| Extra large | 1600 and up | Two or three panes, content capped |

## Worked Example: A Navigation Shell and an Expense Grid

The shell swaps a bottom bar for a rail at 600 dp, keeps its destinations in one list, and caps the content width. The list becomes a grid with no breakpoint at all, because `maxCrossAxisExtent` picks the column count.

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// lib/app_shell.dart
import 'package:material_ui/material_ui.dart';

class Breakpoints {
  static const double medium = 600;
  static const double expanded = 840;
  static const double contentMax = 1100;
}

class AppShell extends StatefulWidget {
  const AppShell({super.key});

  @override
  State<AppShell> createState() => _AppShellState();
}

class _AppShellState extends State<AppShell> {
  int _tab = 0;

  static const _tabs = [
    (icon: Icons.receipt_long, label: 'Expenses'),
    (icon: Icons.pie_chart, label: 'Budget'),
    (icon: Icons.settings, label: 'Settings'),
  ];

  @override
  Widget build(BuildContext context) {
    final compact = MediaQuery.sizeOf(context).width < Breakpoints.medium;
    final body = Center(
      child: ConstrainedBox(
        constraints: const BoxConstraints(maxWidth: Breakpoints.contentMax),
        child: const ExpenseGrid(),
      ),
    );

    if (compact) {
      return Scaffold(
        body: SafeArea(child: body),
        bottomNavigationBar: NavigationBar(
          selectedIndex: _tab,
          onDestinationSelected: (i) => setState(() => _tab = i),
          destinations: [
            for (final t in _tabs)
              NavigationDestination(icon: Icon(t.icon), label: t.label),
          ],
        ),
      );
    }

    return Scaffold(
      body: SafeArea(
        child: Row(
          children: [
            NavigationRail(
              selectedIndex: _tab,
              onDestinationSelected: (i) => setState(() => _tab = i),
              labelType: NavigationRailLabelType.all,
              destinations: [
                for (final t in _tabs)
                  NavigationRailDestination(
                    icon: Icon(t.icon),
                    label: Text(t.label),
                  ),
              ],
            ),
            const VerticalDivider(width: 1),
            Expanded(child: body),
          ],
        ),
      ),
    );
  }
}

class ExpenseGrid extends StatelessWidget {
  const ExpenseGrid({super.key});

  @override
  Widget build(BuildContext context) => GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
          maxCrossAxisExtent: 320,
          mainAxisExtent: 96,
          mainAxisSpacing: 12,
          crossAxisSpacing: 12,
        ),
        itemCount: 40,
        itemBuilder: (context, index) => Card(
          child: ListTile(
            title: Text('Expense $index', overflow: TextOverflow.ellipsis),
          ),
        ),
      );
}
```

Inside a pane, prefer `LayoutBuilder`, because the pane's width depends on what's beside it:

```dart
// lib/expense_detail.dart
import 'package:material_ui/material_ui.dart';

import 'app_shell.dart';

class ExpenseDetail extends StatelessWidget {
  const ExpenseDetail({super.key, required this.sideBySide, required this.stacked});

  final Widget sideBySide;
  final Widget stacked;

  @override
  Widget build(BuildContext context) => LayoutBuilder(
        builder: (context, constraints) =>
            constraints.maxWidth >= Breakpoints.expanded ? sideBySide : stacked,
      );
}
```

## Keeping Content Readable on Large Screens

Long text and forms become hard to read when they stretch across a monitor. Wrap them in `ConstrainedBox` with a `maxWidth` around 600 to 800, inside a `Center`. Convert a `ListView.builder` to `GridView.builder` for cards. Always use the `.builder` constructors for long or unknown lists, so only visible rows are built.

## Orientation, Foldables and Multi-Window

Leave orientation unlocked. A locked orientation on a foldable often gives letterboxing, with the app centred between black bars, and Android's large-screen quality tiers expect both orientations. Android 17 goes further and ignores orientation and resizability restrictions on large screens, so an app that assumes portrait will meet a landscape window there.

If a business rule forces a fixed orientation anyway, read the size from `View.of(context).display.size`, because `MediaQuery` doesn't receive the larger window in compatibility mode.

## Input Beyond Touch

A wide window often means a mouse and keyboard. Add hover states with `MouseRegion` or the built-in `InkWell` hover colour, wire shortcuts with `CallbackShortcuts`, keep visible focus for `Tab`, and keep touch targets at 48 dp at least.

## Verify

```bash
flutter run -d macos     # or -d chrome: drag the window from narrow to wide
```

Resize the window slowly through 600 and 840 and watch for overflow stripes or a jump in scroll position. Then pin the breakpoint with a widget test:

```dart
// test/app_shell_test.dart
import 'package:expense_tracker/app_shell.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  testWidgets('shows a rail at 800 wide', (tester) async {
    tester.view.physicalSize = const Size(800, 600);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.reset);

    await tester.pumpWidget(const MaterialApp(home: AppShell()));

    expect(find.byType(NavigationRail), findsOneWidget);
    expect(find.byType(NavigationBar), findsNothing);
  });
}
```

## Gotchas

1. **`RenderFlex overflowed` only on small phones.** Symptom: yellow and black stripes at 320 dp. Cause: fixed widths in a `Row`. Fix: use `Expanded` or `Flexible`, and test at the narrowest size.
2. **The app is letterboxed on a foldable.** Symptom: black bars either side. Cause: `setPreferredOrientations` locked it. Fix: remove the lock.
3. **`LayoutBuilder` reports an infinite width.** Symptom: `constraints.maxWidth` is `Infinity` inside a horizontal scroller. Cause: the parent gives unbounded space. Fix: move the `LayoutBuilder` above the scroll view.
4. **The whole screen rebuilds when the keyboard opens.** Symptom: jank on focus. Cause: `MediaQuery.of(context)` depends on every metric. Fix: use `MediaQuery.sizeOf` or another aspect-specific getter.
5. **The tab state resets at a breakpoint.** Symptom: the scroll position is lost when the window crosses 600. Cause: the two layouts build different trees. Fix: keep the state above the branch, as `_tab` is here, and reuse the same page keys.

## Anti-Patterns

- `if (Platform.isIOS)` to choose a layout. Width decides.
- A `Container` with a hard-coded width for page content. Cap it with `BoxConstraints`.
- Scaling everything by screen width. Text and touch targets keep their size, and only the layout changes.

## Pre-Merge Checklist

- [ ] Works from 320 dp to 1600 dp with no overflow
- [ ] Layout decisions read width, never device type or orientation
- [ ] Orientation isn't locked, or the reason is written down
- [ ] Long text and forms have a `maxWidth`
- [ ] Mouse, keyboard and focus work on desktop and web

## See Also

- `flutter-fix-layout-issues` for the errors these layouts can throw
- `flutter-design-ui` for the components inside them
- `flutter-add-widget-test` for breakpoint tests
