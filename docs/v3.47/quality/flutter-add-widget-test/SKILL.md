---
name: flutter-add-widget-test
description: Writes fast widget tests with flutter_test that prove one widget's rendering, input handling and state changes without a device - pumping the right frames, finding widgets reliably, faking dependencies, golden files. Use when asked to add a widget test, test a form or a list, cover a button tap, fix a pumpAndSettle timeout, or check what a screen shows.
metadata:
  tags:
  - flutter
  - testing
  - widget-test
---
# Flutter Widget Tests

A widget test builds one widget in a headless test environment and checks what it renders and how it reacts to input. It runs in milliseconds on a developer's machine, so it's the cheapest way to prove UI behaviour, and most of a Flutter app's UI coverage should sit here.

## Step 0: Confirm the Test Setup

Check that the project can run widget tests, and see which version you're on:

```bash
flutter --version
grep -n "flutter_test" -A1 pubspec.yaml
ls test/
```

`flutter_test` belongs under `dev_dependencies` with `sdk: flutter`; `flutter create` adds it for you. Test files live under `test/` and their names must end in `_test.dart`, or the runner skips them.

## Which Test Fits

| You need to prove | Use | Why the others lose here |
|---|---|---|
| A price formatter or a reducer returns the right value | Unit test (`test()`) | A widget test adds a frame loop you don't need |
| A form shows an error after a bad submit | Widget test (`testWidgets()`) | A unit test can't see the tree; an integration test needs a device |
| A chart keeps its exact look | Golden test (`matchesGoldenFile`) | A finder can't see pixels |
| Sign-in reaches the home screen on a real phone | Integration test (see `flutter-add-integration-test`) | A widget test fakes the platform |

## How Pumping Works

The tester doesn't draw frames on its own. Every change you trigger needs a frame before you can assert on the result.

| Call | Does | Reach for it when |
|---|---|---|
| `tester.pumpWidget(child)` | Builds the tree and draws one frame | Starting a test, or swapping the root |
| `tester.pump()` | Draws one frame | After `tester.tap` on a plain `setState` |
| `tester.pump(duration)` | Advances the clock, then draws one frame | Stepping through an animation or a debounce |
| `tester.pumpAndSettle()` | Draws frames until nothing is scheduled | After a dialog opens or a route animates |

`pumpAndSettle` only ends when no frame is scheduled, so it times out on anything that animates forever, such as a `CircularProgressIndicator`.

## Finding Widgets

Prefer the finder a user would describe. `find.text` and `find.bySemanticsLabel` break when the words change, which is usually a real regression. Use `find.byKey` when the same text appears twice, and `find.byType` when the type itself is the point of the check. Narrow a finder with `find.descendant(of: ..., matching: ...)` instead of indexing with `.at(1)`.

| Matcher | Asserts |
|---|---|
| `findsOneWidget` | exactly one match |
| `findsNothing` | no match |
| `findsNWidgets(3)` | exactly three matches |
| `findsAtLeastNWidgets(1)` | at least one match |
| `matchesGoldenFile('name.png')` | the pixels match the stored image |

## Worked Example: An Expense Form

The form rejects an empty amount and passes a valid one to a callback. The widget takes its callback through the constructor, so the test needs no mock.

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// lib/expenses/expense_form.dart
import 'package:material_ui/material_ui.dart';

class ExpenseForm extends StatefulWidget {
  const ExpenseForm({super.key, required this.onSubmit});

  final void Function(double amount) onSubmit;

  @override
  State<ExpenseForm> createState() => _ExpenseFormState();
}

class _ExpenseFormState extends State<ExpenseForm> {
  final _amount = TextEditingController();
  String? _error;

  @override
  void dispose() {
    _amount.dispose();
    super.dispose();
  }

  void _submit() {
    final value = double.tryParse(_amount.text);
    if (value == null || value <= 0) {
      setState(() => _error = 'Enter an amount above zero');
      return;
    }
    setState(() => _error = null);
    widget.onSubmit(value);
  }

  @override
  Widget build(BuildContext context) => Column(
        children: [
          TextField(
            key: const ValueKey('amount'),
            controller: _amount,
            keyboardType: const TextInputType.numberWithOptions(decimal: true),
            decoration: InputDecoration(labelText: 'Amount', errorText: _error),
          ),
          FilledButton(onPressed: _submit, child: const Text('Save')),
        ],
      );
}
```

```dart
// test/expenses/expense_form_test.dart
import 'package:expense_tracker/expenses/expense_form.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

/// Material widgets need a Material ancestor plus theme and direction data.
Widget harness(Widget child) => MaterialApp(home: Scaffold(body: child));

void main() {
  testWidgets('an empty amount shows an error and submits nothing', (tester) async {
    double? submitted;
    await tester.pumpWidget(harness(ExpenseForm(onSubmit: (v) => submitted = v)));

    await tester.tap(find.text('Save'));
    await tester.pump();

    expect(find.text('Enter an amount above zero'), findsOneWidget);
    expect(submitted, isNull);
  });

  testWidgets('a valid amount reaches the callback', (tester) async {
    double? submitted;
    await tester.pumpWidget(harness(ExpenseForm(onSubmit: (v) => submitted = v)));

    await tester.enterText(find.byKey(const ValueKey('amount')), '12.50');
    await tester.tap(find.text('Save'));
    await tester.pump();

    expect(submitted, 12.5);
    expect(find.textContaining('above zero'), findsNothing);
  });

  testWidgets('the form stays usable at a large text scale', (tester) async {
    tester.view.physicalSize = const Size(320, 640);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.reset);

    await tester.pumpWidget(
      MediaQuery(
        data: const MediaQueryData(textScaler: TextScaler.linear(2)),
        child: harness(ExpenseForm(onSubmit: (_) {})),
      ),
    );

    expect(tester.takeException(), isNull);
    expect(find.text('Save'), findsOneWidget);
  });
}
```

## Testing Lists, Gestures and Async State

Long lists build lazily, so an off-screen row isn't in the tree. `await tester.scrollUntilVisible(rowFinder, 300, scrollable: find.byType(Scrollable))` scrolls until the row mounts. For a swipe, `await tester.drag(find.byType(Dismissible), const Offset(-500, 0))` followed by `pumpAndSettle` runs the dismiss animation to the end.

A screen that awaits a `Future` needs two steps: `pump()` shows the loading state, and a second `pump()` after the future completes shows the result. Inject the data source through the constructor or a provider override, and hand it a `Completer` when you need to hold the loading state for an assertion. Real network calls fail in tests by default: `HttpClient` returns status 400 inside `testWidgets`, so fake the client rather than reaching for the network.

## Golden Files Pin the Pixels

A golden test compares a render with a stored image. Create or refresh the image with `flutter test --update-goldens`, then commit it. Rendering differs between operating systems, so generate goldens on the same platform your CI runs. Since Flutter 3.32 the default comparator for `integration_test` differs, so keep goldens in `test/`.

```dart
// test/expenses/expense_form_golden_test.dart
import 'package:expense_tracker/expenses/expense_form.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  testWidgets('the form matches its golden', (tester) async {
    await tester.pumpWidget(
      MaterialApp(home: Scaffold(body: ExpenseForm(onSubmit: (_) {}))),
    );

    await expectLater(find.byType(ExpenseForm), matchesGoldenFile('goldens/expense_form.png'));
  });
}
```

## Verify

```bash
flutter test test/expenses/expense_form_test.dart      # one file
flutter test --name "valid amount"                     # one test by name
flutter test --coverage                                # writes coverage/lcov.info
```

Then break the widget on purpose, for instance by removing the validation, and watch the test fail. A test that can't fail proves nothing.

## Gotchas

1. **`No Material widget found`.** Symptom: a red exception naming `TextField` or `ListTile`. Cause: the widget was pumped bare, without a Material ancestor. Fix: wrap it in `MaterialApp` plus `Scaffold`.
2. **`pumpAndSettle timed out`.** Symptom: the test fails after 10 minutes of fake time. Cause: an endless animation, usually a spinner, keeps scheduling frames. Fix: use `pump(const Duration(seconds: 1))`, or end the loading state before settling.
3. **`Found 0 widgets with text`.** Symptom: an expectation fails right after a tap. Cause: no frame ran after the state change. Fix: add `await tester.pump()`.
4. **A row is missing from a long list.** Symptom: `find.text` fails only for later rows. Cause: `ListView.builder` hasn't mounted that row. Fix: `scrollUntilVisible` first.
5. **`A Timer is still pending`.** Symptom: the test fails at the end with a pending timer. Cause: a debounce or delayed call outlived the test. Fix: pump past its duration, or cancel it in `dispose`.
6. **State leaks between tests.** Symptom: tests pass alone and fail together. Cause: a global singleton kept its value. Fix: reset it in `tearDown`, or inject it.

## Anti-Patterns

- Asserting on private state. Assert on what the user sees.
- Sleeping with `Future.delayed` in a test. Advance the fake clock with `pump(duration)`.
- Putting several behaviours in one `testWidgets`. Write one behaviour per test, so a failure names the bug.
- Faking the widget under test. Fake only its collaborators.

## Pre-Merge Checklist

- [ ] Each new widget state has a test: empty, loading, error and filled
- [ ] Every test passes `flutter test` and fails when the behaviour is broken
- [ ] No real network, disk or platform channel is reached
- [ ] Timers and animations end within the test
- [ ] Golden images were generated on the CI platform

## See Also

- `flutter-add-integration-test` for flows on a real device
- `dart-unit-test` for plain Dart logic and mocks
- `flutter-fix-layout-issues` for overflow failures a test exposes
