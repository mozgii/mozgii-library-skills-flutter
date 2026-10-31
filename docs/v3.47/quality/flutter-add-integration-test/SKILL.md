---
name: flutter-add-integration-test
description: Adds integration tests that drive a whole Flutter app on a real device, emulator or browser with the integration_test package, explores the running app first through the Dart and Flutter MCP server, runs the suite locally and in Firebase Test Lab, and captures performance timelines. Use when asked to add an end-to-end test, test a login or checkout flow on a device, automate a user journey, or turn an MCP session into a permanent test.
metadata:
  tags:
  - flutter
  - testing
  - integration-test
---
# Flutter Integration Tests

An integration test runs your real app in a real engine, on a phone, an emulator or Chrome, and walks through a user journey. It's slow, so keep a small number of them for the flows that cost money when they break: sign-in, purchase, onboarding.

## Step 0: Check the Project and a Target

```bash
flutter --version
flutter devices
grep -n "integration_test" -A1 pubspec.yaml
```

Add the two SDK packages when they're missing:

```bash
flutter pub add 'dev:integration_test:{"sdk":"flutter"}'
flutter pub add 'dev:flutter_test:{"sdk":"flutter"}'
```

## Choosing the Runner

| Target | Command | Note |
|---|---|---|
| Local phone or emulator, everyday | `flutter test integration_test/` | Simplest; prints results in the terminal |
| One device out of several | `flutter test integration_test/checkout_test.dart -d <device-id>` | Take the id from `flutter devices` |
| Chrome | `flutter drive --driver=test_driver/integration_test.dart --target=integration_test/checkout_test.dart -d chrome` | Start `chromedriver --port=4444` first |
| Headless web in CI | The same `flutter drive` command with `-d web-server` | `chromedriver` is still required |
| Performance numbers | `flutter drive` with a profile build and a perf driver | `flutter test` can't write timelines |
| Many physical devices | Firebase Test Lab | See the section below |

`flutter drive` needs a small host-side driver file; `flutter test` doesn't.

```dart
// test_driver/integration_test.dart
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() => integrationDriver();
```

## Explore the Live App Through the MCP Server

An agent connected to the Dart and Flutter MCP server can try a flow on the running app before writing it down, which replaces guessing at keys and text with reading the real tree. `launch_app` is off by default, so start the server with `dart mcp-server --enable flutter_app_lifecycle` (see `flutter-agent-tooling`). The driver commands also need the app built with the Flutter Driver extension:

```dart
// lib/main_test.dart: an entry point only for exploration
import 'package:flutter_driver/driver_extension.dart';
import 'package:expense_tracker/main.dart' as app;

void main() {
  enableFlutterDriverExtension();
  app.main();
}
```

1. Call `launch_app` with `target: "lib/main_test.dart"`; it returns the DTD URI.
2. Connect with the `dtd` tool: command `listDtdUris`, then `connect`.
3. Call `widget_inspector` with command `get_widget_tree` to see the real keys, text and types. Never guess a selector.
4. Call `flutter_driver_command` with `tap`, `enter_text`, `scroll` or `scrollIntoView`. Use `waitFor`, `waitForAbsent` or `get_health` after anything that navigates or animates. A row missing from a lazy list needs `scrollIntoView` before you touch it.
5. Write each confirmed step into a test, then remove `lib/main_test.dart` from anything you ship.

## Worked Example: Adding an Expense End to End

Give critical widgets a stable key, such as `ValueKey('add_expense')`, so the test survives a copy change.

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// integration_test/add_expense_test.dart
import 'package:expense_tracker/main.dart' as app;
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('expenses', () {
    testWidgets('a new expense appears in the list and in the total', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.tap(find.byKey(const ValueKey('add_expense')));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(const ValueKey('amount')), '19.99');
      await tester.enterText(find.byKey(const ValueKey('note')), 'Train ticket');
      await tester.tap(find.byKey(const ValueKey('save_expense')));
      await tester.pumpAndSettle();

      final row = find.text('Train ticket');
      await tester.scrollUntilVisible(
        row,
        200,
        scrollable: find.byType(Scrollable).first,
      );
      expect(row, findsOneWidget);
      expect(find.textContaining('19.99'), findsWidgets);

      await binding.takeScreenshot('expense_added');
    });
  });
}
```

On Android, `takeScreenshot` needs `await binding.convertFlutterSurfaceToImage()` before the first capture.

## Firebase Test Lab on Android

Test Lab needs two APKs. Build them from the `android/` folder, then upload both in the console or with `gcloud`:

```bash
flutter build apk --debug
cd android
./gradlew app:assembleAndroidTest
./gradlew app:assembleDebug -Ptarget=../integration_test/add_expense_test.dart
cd ..
gcloud firebase test android run --type instrumentation \
  --app build/app/outputs/flutter-apk/app-debug.apk \
  --test build/app/outputs/apk/androidTest/debug/app-debug-androidTest.apk
```

The instrumentation runner must be set up in `android/app/src/androidTest/` first; the `integration_test` package README has the Java test class to copy. iOS uses `xcodebuild build-for-testing`, then `gcloud firebase test ios run`.

## Capturing a Performance Timeline

Wrap the interesting actions in `binding.traceAction`, then read the result in a driver that writes the summary to disk. Run it against a profile build, because debug timings are meaningless.

```dart
// integration_test/scroll_performance_test.dart
import 'package:expense_tracker/main.dart' as app;
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('the expense list scrolls', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await binding.traceAction(() async {
      await tester.fling(find.byType(ListView), const Offset(0, -600), 3000);
      await tester.pumpAndSettle();
    }, reportKey: 'expense_scroll');
  });
}
```

```dart
// test_driver/perf_driver.dart
import 'package:flutter_driver/flutter_driver.dart' as driver;
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() => integrationDriver(
      responseDataCallback: (data) async {
        final trace = data?['expense_scroll'] as Map<String, dynamic>?;
        if (trace == null) return;
        final summary = driver.TimelineSummary.summarize(driver.Timeline.fromJson(trace));
        await summary.writeTimelineToFile('expense_scroll', pretty: true, includeSummary: true);
      },
    );
```

```bash
flutter drive --profile --driver=test_driver/perf_driver.dart --target=integration_test/scroll_test.dart
```

## Verify

The examples were compiled, not run on a device, so run the suite on your own target before you trust it.

```bash
flutter test integration_test/ -d <device-id>      # passes on a device you can watch
flutter analyze integration_test test_driver        # no missing imports
```

Run the suite twice in a row. A test that passes once and fails once is a race, usually a missing `pumpAndSettle` or real network timing.

## Gotchas

1. **`PumpAndSettleTimedOutException`.** Symptom: the test stalls on a screen with a spinner. Cause: an endless animation schedules frames forever. Fix: pump for a fixed duration, or wait for a finder in a short loop.
2. **`The finder could not find any widget` on a list row.** Symptom: the row exists when you scroll by hand. Cause: lazy lists mount rows on demand. Fix: `scrollUntilVisible` first.
3. **`Failed to connect to chromedriver`.** Symptom: `flutter drive -d chrome` exits at once. Cause: `chromedriver` isn't running, or its major differs from Chrome's. Fix: start the matching version with `--port=4444`.
4. **A test passes alone and fails in the folder.** Symptom: the second file sees old data. Cause: each file shares the app's storage on the device. Fix: clear storage in `setUp`, or use a test account per file.
5. **The app calls production.** Symptom: test data lands in real analytics. Cause: the test ran `main()` with production config. Fix: pass `--dart-define=ENV=test` and point the app at a staging backend.
6. **`enableFlutterDriverExtension` shipped.** Symptom: a release build exposes driver commands. Cause: it was added to `lib/main.dart`. Fix: keep it in a separate entry point.

## Anti-Patterns

- Testing validation rules here. Cover them in widget tests, where they run in milliseconds.
- Calling `Future.delayed` to wait for the UI. Wait on a finder.
- Selecting by list index. Select by key or text.
- Mixing the legacy `flutter_driver` API (`driver.tap`, `driver.waitFor`) into new tests. Keep it only in old suites you're migrating, and write new tests with `WidgetTester`.

## Pre-Merge Checklist

- [ ] The test covers one user journey and is named after it
- [ ] Critical widgets carry a `ValueKey`
- [ ] The suite passed twice in a row on a real target
- [ ] The app under test talks to a test backend
- [ ] No driver extension reaches a release entry point

## See Also

- `flutter-add-widget-test` for faster UI checks
- `flutter-agent-tooling` for the MCP server setup
- `flutter-performance` for reading timelines
