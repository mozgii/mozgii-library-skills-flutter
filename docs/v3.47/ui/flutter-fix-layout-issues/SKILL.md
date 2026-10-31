---
name: flutter-fix-layout-issues
description: Diagnoses and fixes Flutter layout errors from their exact message - RenderFlex overflowed, Vertical viewport was given unbounded height, InputDecorator unbounded width, Incorrect use of ParentData widget, RenderBox was not laid out - using the widget inspector and the Dart and Flutter MCP tools. Use when the screen shows yellow and black stripes, a red error screen, a blank list, or a layout exception in the console.
metadata:
  tags:
  - flutter
  - ui
  - layout
  - debugging
---
# Fixing Flutter Layout Errors

Nearly every Flutter layout error comes from one negotiation going wrong: a parent passes constraints down, a child answers with a size, and the parent places it. Once you know which side broke the deal, the fix is usually one wrapper widget. This skill maps each error message to its cause.

## Step 0: Capture the First Error

```bash
flutter --version
flutter run -d <device-id>      # debug mode, so the full exception prints
```

Read from the top of the console. Flutter prints the first failure with a widget chain, then a cascade of follow-on errors. The first one is the one to fix. The examples also import `material_ui: ^1.9.0`.

## The Error Map

| Message contains | What happened | Usual fix |
|---|---|---|
| `RenderFlex overflowed by N pixels` | A `Row` or `Column` child wants more room than the parent has | Wrap that child in `Expanded` or `Flexible` |
| `Vertical viewport was given unbounded height` | A scrollable sits in a parent with infinite height, usually a `Column` | Wrap the scrollable in `Expanded`, or give it a `SizedBox` height |
| `Horizontal viewport was given unbounded width` | The same, sideways, usually in a `Row` | Wrap it in `Expanded`, or set a width |
| `An InputDecorator...cannot have an unbounded width` | A `TextField` sits directly in a `Row` | Wrap the field in `Expanded` or `Flexible` |
| `Incorrect use of ParentData widget` | `Expanded` is outside a `Flex`, or `Positioned` is outside a `Stack` | Make it a direct child of the right parent |
| `RenderBox was not laid out` | A symptom of another failure above it | Ignore it and fix the earlier error |
| `BoxConstraints forces an infinite height` | A child asked for `double.infinity` inside an unbounded parent | Remove the infinite size, or bound the parent |

## Why Each Wrapper Works

`Column` gives its children unbounded height so it can stack them. A `ListView` asks for all the height it can get, so the two together ask for infinity. `Expanded` turns the leftover space into a tight, finite constraint, and the list then has a real size.

`Expanded` and `Flexible` differ in one way. `Expanded` forces the child to fill its share. `Flexible` allows the child to be smaller and leaves the rest unused, which suits text that may be short.

| Situation | Choose | Why the others lose here |
|---|---|---|
| The child should use all remaining space | `Expanded` | `Flexible` can leave a gap |
| The child should be only as big as its content, up to the limit | `Flexible` | `Expanded` stretches it |
| The child has a known fixed size | `SizedBox` | `Expanded` ignores the design value |
| Long text in a tight row | `Expanded` plus `TextOverflow.ellipsis` | `FittedBox` shrinks the type below readable |
| Content is taller than the screen | `SingleChildScrollView` around the `Column` | `Expanded` can't create room that doesn't exist |
| Children must wrap onto a new line | `Wrap` | A `Row` never wraps |

## Worked Example: A Receipt Screen With Three Bugs

This screen throws two errors as soon as it renders, and a third once a real note arrives. It shows these receipts:

```dart
// lib/receipts/receipt.dart
class Receipt {
  const Receipt(this.longMerchantNote);
  final String longMerchantNote;
}
```

```dart
// lib/receipts/broken_receipt_screen.dart
import 'package:material_ui/material_ui.dart';

import 'receipt.dart';

class BrokenReceiptScreen extends StatelessWidget {
  const BrokenReceiptScreen({super.key, this.receipts = const [], this.search});

  final List<Receipt> receipts;
  final TextEditingController? search;

  @override
  Widget build(BuildContext context) => Column(
        children: [
          const Text('Receipts'),
          ListView.builder(                       // unbounded height
            itemCount: receipts.length,
            itemBuilder: (context, i) => Row(
              children: [
                const Icon(Icons.receipt),
                Text(receipts[i].longMerchantNote), // overflows on a phone
              ],
            ),
          ),
          Row(
            children: [
              const Icon(Icons.search),
              TextField(controller: search),        // unbounded width
            ],
          ),
        ],
      );
}
```

```dart
// lib/receipts/receipt_screen.dart
import 'package:material_ui/material_ui.dart';

import 'receipt.dart';

class ReceiptScreen extends StatelessWidget {
  const ReceiptScreen({super.key, this.receipts = const [], this.search});

  final List<Receipt> receipts;
  final TextEditingController? search;

  @override
  Widget build(BuildContext context) => Column(
        children: [
          const Text('Receipts'),
          Expanded(
            child: ListView.builder(
              itemCount: receipts.length,
              itemBuilder: (context, i) => Row(
                children: [
                  const Icon(Icons.receipt),
                  const SizedBox(width: 8),
                  Expanded(
                    child: Text(
                      receipts[i].longMerchantNote,
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                  ),
                ],
              ),
            ),
          ),
          Row(
            children: [
              const Icon(Icons.search),
              const SizedBox(width: 8),
              Expanded(child: TextField(controller: search)),
            ],
          ),
        ],
      );
}
```

## The List Inside a Scroll View

A list inside another scroll view is the harder variant, because `Expanded` isn't available there. Don't set `shrinkWrap: true` on a long list: it lays out every row up front, which defeats lazy building. Move to one `CustomScrollView` with slivers instead, so everything scrolls together and rows stay lazy.

```dart
// lib/receipts/receipt_feed.dart
import 'package:material_ui/material_ui.dart';

import 'receipt.dart';

class ReceiptFeed extends StatelessWidget {
  const ReceiptFeed({super.key, required this.summary, required this.receipts});

  final Widget summary;
  final List<Receipt> receipts;

  @override
  Widget build(BuildContext context) => CustomScrollView(
        slivers: [
          SliverToBoxAdapter(child: summary),
          SliverList.builder(
            itemCount: receipts.length,
            itemBuilder: (context, i) =>
                ListTile(title: Text(receipts[i].longMerchantNote)),
          ),
        ],
      );
}
```

`shrinkWrap` is fine for a short list of a few fixed rows, typically with `physics: const NeverScrollableScrollPhysics()`.

## The Keyboard Overflow

When the keyboard opens, `Scaffold` shrinks its body, and a `Column` of fixed-height fields overflows at the bottom. Put the form in a `SingleChildScrollView`, which keeps `resizeToAvoidBottomInset` working. Setting that flag to `false` hides the overflow and leaves the focused field behind the keyboard.

## Working With an Agent Through the MCP Server

An agent connected to the Dart and Flutter MCP server can read the live app instead of guessing. Use `get_runtime_errors` for the exception text, and `widget_inspector` with `get_widget_tree` to see the real tree around the failing widget. After a fix, call `hot_reload`, then `get_runtime_errors` again; `hot_restart` is needed when the change touches `initState` or global state. See `flutter-agent-tooling` for setup.

## Verify

Without an agent, the Flutter Inspector in DevTools shows the constraints at each level, and its Layout Explorer shows how a `Row` or `Column` shares space. Add `debugPaintSizeEnabled = true` temporarily to outline every box. Check the fix at the narrowest window you support with the largest text scale, because that's where overflows hide.

```dart
// test/receipts/receipt_screen_test.dart
import 'package:expense_tracker/receipts/receipt_screen.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  testWidgets('receipts fit a small phone at large text', (tester) async {
    tester.view.physicalSize = const Size(320, 568);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.reset);

    await tester.pumpWidget(
      const MediaQuery(
        data: MediaQueryData(textScaler: TextScaler.linear(1.5)),
        child: MaterialApp(home: Scaffold(body: ReceiptScreen())),
      ),
    );

    expect(tester.takeException(), isNull);
  });
}
```

## Gotchas

1. **The list is blank, with no error in release.** Symptom: a grey box in a release build, and a red screen in debug. Cause: a layout exception. Fix: always debug layout in debug mode.
2. **`Expanded` inside a `ListView` row throws.** Symptom: `RenderFlex children have non-zero flex but incoming height constraints are unbounded`. Cause: `Expanded` in a `Column` inside a scrollable has nothing to divide. Fix: remove it, or give the `Column` a fixed height.
3. **Fixing one overflow makes a new one appear.** Symptom: the stripes move to the next widget. Cause: more than one fixed-size child in the row. Fix: make every text child flexible.
4. **`Expanded` was wrapped in `Padding`.** Symptom: `Incorrect use of ParentData widget`. Cause: `Expanded` must be a direct child of the `Flex`. Fix: put the `Padding` inside the `Expanded`.
5. **The error only appears for some users.** Symptom: reports from one language or from accessibility users. Cause: longer strings or a larger text scale. Fix: test with a long locale and a 2.0 scale.

## Anti-Patterns

- Wrapping everything in `SingleChildScrollView` to silence an overflow. Find which child is too big.
- Setting `shrinkWrap: true` on long lists. Use slivers.
- Hard-coding heights from one test phone. Let flex widgets divide the space.
- Reaching for `FittedBox` for body text. It shrinks type below a readable size.

## Pre-Merge Checklist

- [ ] The first error was fixed, and the cascade is gone
- [ ] No hard-coded size replaced a flex widget
- [ ] Checked at 320 dp wide, and at a 1.5 to 2.0 text scale
- [ ] Long lists remain lazy, with no `shrinkWrap`
- [ ] A widget test pins the case that failed

## See Also

- `flutter-build-responsive-layout` for designing layouts that don't break
- `flutter-performance` for list costs
- `flutter-agent-tooling` for the runtime error tools
