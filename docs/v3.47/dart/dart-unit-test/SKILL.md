---
name: dart-unit-test
description: Writes Dart unit tests with package:test or flutter_test - groups, setUp, async code, streams, fake time - fakes and mocks with mockito or mocktail, package:checks assertions, and coverage reports in LCOV. Use when asked to add a unit test, mock a dependency, generate mocks with build_runner, migrate expect to checks, collect test coverage, or fix a MissingStubError.
metadata:
  tags:
  - dart
  - testing
  - mocking
  - coverage
---
# Dart Unit Tests

Unit tests are the fastest feedback a Dart or Flutter project has: thousands run in seconds, with no device. They check logic in isolation, so a failure points at one function. This skill covers structure, test doubles, assertion style and coverage.

## Step 0: Find the Test Stack

```bash
dart --version
grep -nE "^\s+(test|flutter_test|mockito|mocktail|checks|coverage|fake_async):" pubspec.yaml
ls test
```

A Flutter project imports `package:flutter_test/flutter_test.dart`, which re-exports the test API. A pure Dart package imports `package:test/test.dart`. Run with `flutter test` or `dart test` to match. The examples also import `mocktail: ^1.0.5`, `test: ^1.33.0`.

## Choosing a Test Double

| Need | Choose | Why the others lose here |
|---|---|---|
| A collaborator with simple behaviour | A hand-written fake | No package, no build step, and it reads like real code |
| Check that a call happened, in a project with no codegen | `mocktail` | Mockito needs `build_runner` |
| Strict typing, generated stubs, a project already on `build_runner` | `mockito` with `@GenerateNiceMocks` | Mocktail needs `registerFallbackValue` for custom types |
| An HTTP layer | `MockClient` from `package:http/testing.dart` | A mock of `Client` repeats what `MockClient` already gives you |
| Code driven by time | `fake_async` | Real delays make tests slow and flaky |

Inject every outside dependency through the constructor, since a class that builds its own `http.Client` can't be tested. Represent URLs as `Uri` values, not strings.

## Structure

Mirror `lib/`: `lib/src/budget.dart` is tested in `test/src/budget_test.dart`. Names must end in `_test.dart`. `group` gives shared context, `setUp` runs before each test, and `tearDown` after. `setUpAll` runs once for the group, for expensive work. A test can be `async`, and the runner waits for the `Future`. Integration tests sit in `integration_test/`, which the default runner ignores, so name the folder when you run them.

## Worked Example: Testing a Budget Service

```dart
// lib/src/budget_service.dart
abstract interface class RatesApi {
  Future<double> rate(String from, String to);
}

class OverBudget implements Exception {
  const OverBudget(this.excessCents);
  final int excessCents;
}

class BudgetService {
  BudgetService(this._rates, {required this.limitCents});

  final RatesApi _rates;
  final int limitCents;

  Future<int> spend(int cents, {required String currency}) async {
    final rate = currency == 'USD' ? 1.0 : await _rates.rate(currency, 'USD');
    final usd = (cents * rate).round();
    if (usd > limitCents) throw OverBudget(usd - limitCents);
    return limitCents - usd;
  }
}
```

### With a Fake and package:test

```dart
// test/src/budget_service_test.dart
import 'package:budget/src/budget_service.dart';
import 'package:test/test.dart';

class FixedRates implements RatesApi {
  FixedRates(this.value);
  final double value;
  int calls = 0;

  @override
  Future<double> rate(String from, String to) async {
    calls++;
    return value;
  }
}

void main() {
  group('BudgetService.spend', () {
    late FixedRates rates;
    late BudgetService service;

    setUp(() {
      rates = FixedRates(2);
      service = BudgetService(rates, limitCents: 1000);
    });

    test('dollars need no rate lookup', () async {
      expect(await service.spend(400, currency: 'USD'), 600);
      expect(rates.calls, 0);
    });

    test('other currencies convert first', () async {
      expect(await service.spend(400, currency: 'EUR'), 200);
    });

    test('throws with the excess when over the limit', () {
      expect(
        service.spend(600, currency: 'EUR'),
        throwsA(isA<OverBudget>().having((e) => e.excessCents, 'excess', 200)),
      );
    });
  });
}
```

Don't `await` a future you pass to `throwsA`, because the matcher handles it. Streams use `expectLater(stream, emitsInOrder([1, 2, emitsDone]))`.

### With Mockito and build_runner

```bash
dart pub add dev:mockito dev:build_runner
```

```dart
// test/src/budget_service_mockito_test.dart
import 'package:budget/src/budget_service.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:test/test.dart';

@GenerateNiceMocks([MockSpec<RatesApi>()])
import 'budget_service_mockito_test.mocks.dart';

void main() {
  test('asks for the rate once', () async {
    final rates = MockRatesApi();
    when(rates.rate('EUR', 'USD')).thenAnswer((_) async => 2);

    final left = await BudgetService(rates, limitCents: 1000).spend(100, currency: 'EUR');

    expect(left, 800);
    verify(rates.rate('EUR', 'USD')).called(1);
  });
}
```

```bash
dart run build_runner build
```

`@GenerateNiceMocks` returns a simple default for unstubbed calls, so prefer it to `@GenerateMocks`. Use `thenReturn` for sync values and always `thenAnswer` for a `Future` or a `Stream`. Argument matchers are `any`, `anyNamed('x')` and `captureAny`.

### With Mocktail

```dart
// test/src/budget_service_mocktail_test.dart
import 'package:budget/src/budget_service.dart';
import 'package:mocktail/mocktail.dart';
import 'package:test/test.dart';

class MockRatesApi extends Mock implements RatesApi {}

void main() {
  test('asks for the rate once', () async {
    final rates = MockRatesApi();
    when(() => rates.rate(any(), any())).thenAnswer((_) async => 2);

    await BudgetService(rates, limitCents: 1000).spend(100, currency: 'EUR');

    verify(() => rates.rate('EUR', 'USD')).called(1);
  });
}
```

Custom types used with `any()` need `registerFallbackValue` in `setUpAll`.

## Assertions With package:checks

`package:checks` replaces `expect` matchers with typed methods, so wrong assertions fail at compile time and completion lists what's available. It's still pre-1.0 (0.3.x), so weigh that before migrating a large suite.

```dart
// test/src/remaining_test.dart
import 'package:checks/checks.dart';
import 'package:test/scaffolding.dart';

class User {
  const User(this.name, this.age);
  final String name;
  final int age;
}

void main() {
  test('remaining budget', () {
    const remaining = 600;
    const categories = ['food', 'travel'];
    const user = User('Ada', 30);

    check(remaining).equals(600);
    check(categories).deepEquals(['food', 'travel']);
    check(user)
      ..has((u) => u.name, 'name').equals('Ada')
      ..has((u) => u.age, 'age').isGreaterThan(18);
  });
}
```

Async checks must be awaited, or a failure is lost: `await check(future).throws<OverBudget>()`. The full mapping is in `references/checks-migration.md`.

## Coverage Reports

```bash
flutter test --coverage                     # Flutter: writes coverage/lcov.info
dart pub add dev:coverage
dart run coverage:test_with_coverage        # Dart: coverage.json and lcov.info in coverage/
```

In a workspace, name the folders: `dart run coverage:test_with_coverage -- pkgs/foo/test pkgs/bar/test`. For branch and function data, pass `--branch-coverage` and `--function-coverage`.

Exclude code a test can't reach with comments: `// coverage:ignore-line`, `// coverage:ignore-start` with `// coverage:ignore-end`, and `// coverage:ignore-file`. The manual path uses `dart run coverage:format_coverage --lcov --check-ignore`, and `--check-ignore` is what makes those comments count.

## Verify

```bash
dart test                                 # or flutter test
dart test --name "over the limit"         # one test
dart test -p chrome                       # code that also runs on web
```

Then change the code under test and confirm a test fails. Then read `coverage/lcov.info` for the branches you meant to cover.

## Gotchas

1. **`MissingStubError`.** Symptom: an unstubbed mock call throws. Cause: `@GenerateMocks`, which is strict. Fix: use `@GenerateNiceMocks`, or stub the call.
2. **`Invalid argument: thenReturn with a Future`.** Symptom: an `ArgumentError` from `when`. Cause: `thenReturn` on an async method. Fix: `thenAnswer((_) async => value)`.
3. **`Target of URI doesn't exist: *.mocks.dart`.** Symptom: an analyzer error. Cause: `build_runner` hasn't run, or the import name doesn't match the file. Fix: run it, and match the test file's name.
4. **A test passes though the code is broken.** Symptom: a failing assertion goes unnoticed. Cause: an un-awaited `expectLater` or `check`. Fix: `await` it, and enable the `unawaited_futures` lint.
5. **`A test tried to use any or captureAny on a parameter of type`.** Symptom: mocktail fails on `any()`. Cause: a custom type has no fallback. Fix: call `registerFallbackValue` in `setUpAll`.
6. **Tests depend on order.** Symptom: they fail with `dart test --test-randomize-ordering-seed=random`. Cause: shared mutable state. Fix: build fresh objects in `setUp`.

## Anti-Patterns

- Mocking the class under test. Mock only its collaborators.
- Verifying every call. Assert on results, and verify only side effects that matter.
- Using `sleep` or real delays. Use `fake_async`.
- Chasing 100% coverage with assertion-free tests. A covered line isn't a tested line.

## Pre-Merge Checklist

- [ ] New logic has tests for the happy path, the edge cases and the failure
- [ ] Each test has one reason to fail
- [ ] Nothing touches the real network, disk or clock
- [ ] Every async assertion is awaited
- [ ] Coverage didn't fall

## See Also

- `flutter-add-widget-test` for UI tests
- `flutter-apply-architecture-best-practices` for testable layers
- `dart-analyze-and-fix` for the `unawaited_futures` lint
