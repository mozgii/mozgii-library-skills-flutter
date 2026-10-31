---
title: "Flutter Code Quality Standards"
author: MOZGII, Inc.
lang: en
kind: rule
description: "very_good_analysis lint config, testing pyramid (unit/widget/integration), coverage thresholds (75% overall), code review checklist, CI quality gate, prohibited code patterns, Crashlytics non-fatal reporting standards, streaming provider test coverage, and golden tests for design system components."
tags: ["flutter", "quality", "testing", "linting", "coverage", "ci", "crashlytics", "golden-tests", "streaming"]
---
# Flutter Code Quality Standards

Applies to all Dart source in `lib/` and all tests in `test/`. Governs lint configuration, testing pyramid, coverage thresholds, code review checklist, and CI quality gates. Every Flutter project must meet these standards before any production release.

---

## Lint Configuration

Use `very_good_analysis`: the strictest and most comprehensive Flutter lint ruleset.

```yaml
# pubspec.yaml
dev_dependencies:
  very_good_analysis: ^11.0.0
```

```yaml
# analysis_options.yaml
include: package:very_good_analysis/analysis_options.yaml

linter:
  rules:
    # Project-specific overrides (add only what is justified)
    avoid_print: true
    prefer_const_constructors: true
    prefer_const_declarations: true
    prefer_final_locals: true
    unnecessary_nullable_for_final_variable_declarations: true
```

Run `dart analyze` with zero warnings before any commit. Warnings are errors in CI.

---

## Testing Pyramid

Every Flutter project must maintain all three test layers.

```
test/
  unit/                   # fast, no Flutter, no IO
    features/
      auth/
        domain/
          sign_in_usecase_test.dart
        data/
          auth_repository_impl_test.dart
  widget/                 # Flutter widget rendering tests
    features/
      auth/
        presentation/
          login_page_test.dart
          auth_button_test.dart
  integration/            # real device / emulator, full flows
    auth_flow_test.dart
    onboarding_flow_test.dart
```

### Unit test requirements

- Every use case: tested with success and every failure path
- Every repository implementation: tested with mock data sources
- Every domain entity method: tested with edge cases
- Every utility function: tested with edge cases and null inputs

```dart
// test/features/auth/domain/sign_in_usecase_test.dart
import 'package:app/core/errors/failure.dart';
import 'package:app/features/auth/domain/entities/user.dart';
import 'package:app/features/auth/domain/repositories/auth_repository.dart';
import 'package:app/features/auth/domain/usecases/sign_in_usecase.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fpdart/fpdart.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

// Unit test pattern — mocktail, no @GenerateMocks annotation needed
void main() {
  const tUser = User(id: '1', email: 'a@example.com', displayName: 'A');
  const tParams = SignInParams(email: 'a@example.com', password: 'pw');
  late MockAuthRepository mockRepo;
  late SignInUseCase useCase;

  setUp(() {
    mockRepo = MockAuthRepository();
    useCase = SignInUseCase(mockRepo);
    registerFallbackValue(const SignInParams(email: '', password: ''));
  });

  group('SignInUseCase', () {
    test('calls repository with correct params', () async {
      when(() => mockRepo.signIn(any(), any()))
          .thenAnswer((_) async => right(tUser));

      await useCase(tParams);

      verify(() => mockRepo.signIn(tParams.email, tParams.password)).called(1);
    });

    test('returns Failure on network error', () async {
      when(() => mockRepo.signIn(any(), any()))
          .thenAnswer((_) async => left(const Failure.network('timeout')));

      final result = await useCase(tParams);

      expect(result.isLeft(), isTrue);
    });
  });
}
```

### Widget test requirements

- Every custom widget with 3+ visual states must have a widget test per state
- Every page must have a smoke test (renders without throwing)
- Mock Riverpod providers with `ProviderScope(overrides: [someProvider.overrideWith(...)])`: `overrideWith` belongs to the provider, and `ProviderScope` takes the list

```dart
// test/features/auth/presentation/login_page_test.dart
import 'package:app/features/auth/presentation/login_page.dart';
import 'package:app/features/auth/presentation/providers/auth_provider.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

class MockAuthNotifier extends AuthNotifier {}

// Widget test pattern
void main() {
  testWidgets('LoginPage shows error on invalid credentials', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          authProvider.overrideWith(() => MockAuthNotifier()),
        ],
        child: const MaterialApp(home: LoginPage()),
      ),
    );

    await tester.enterText(find.byKey(const Key('email_field')), 'bad@email');
    await tester.tap(find.byKey(const Key('sign_in_button')));
    await tester.pump();

    expect(find.text('Invalid email format'), findsOneWidget);
  });
}
```

### Integration test requirements

- Minimum: happy-path auth flow (sign in → home screen)
- Minimum: primary feature flow (the core user journey)
- Run on physical device or emulator: never mock network in integration tests

---

## Coverage Thresholds

| Layer | Minimum coverage |
|---|---|
| `domain/` (entities, use cases) | 90% |
| `data/` (repository implementations) | 80% |
| `presentation/` (widgets, pages) | 60% |
| `core/` (errors, utils) | 85% |
| Overall project | 75% |

Check coverage:
```bash
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html
```

---

## Code Review Checklist

Before approving any PR:

```
[ ] dart analyze returns 0 warnings
[ ] flutter test passes with no failures
[ ] Coverage thresholds met (run flutter test --coverage)
[ ] No hardcoded colors, sizes, or strings (must use theme/constants)
[ ] No business logic in build() methods
[ ] All async methods handle errors (Either<Failure,T> or try/catch)
[ ] AnimationController.dispose() called in dispose()
[ ] Stream subscriptions cancelled in dispose()
[ ] No print() in production code
[ ] New features have unit tests for happy path and at least 2 error paths
[ ] New widgets have widget smoke tests
[ ] No commented-out code
[ ] No TODO comments without an associated issue number
```

---

## CI Quality Gate

Add to CI pipeline (GitHub Actions / Codemagic / Bitrise). Keep `flutter-version` on the release your project builds with; `subosito/flutter-action` reads `3.47.x` as the latest 3.47 stable. A release older than your packages' SDK floor fails at `flutter pub get`.

```yaml
# .github/workflows/flutter_quality.yml
name: Flutter Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.47.x'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Run code generation
        run: dart run build_runner build

      - name: Analyze
        run: flutter analyze --fatal-infos --fatal-warnings

      - name: Test with coverage
        run: flutter test --coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep 'lines' | awk '{print $2}' | tr -d '%')
          if (( $(echo "$COVERAGE < 75" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 75% threshold"
            exit 1
          fi
```

---

## Prohibited Code Patterns

| Pattern | Reason | Fix |
|---|---|---|
| `print(anything)` in lib/ | Leaks to production logs | Use `debugPrint()` + `kDebugMode` guard |
| `dynamic` without justification | Defeats type safety | Use generics or sealed types |
| `//TODO` without issue number | Untracked debt | `// TODO(#123): description` |
| `setState` at widget tree root | Full subtree rebuild | Use Riverpod select() |
| Empty catch blocks `catch(_) {}` | Swallows errors silently | Log or rethrow |
| `as Type` cast without check | Runtime crash | Use `is` check or pattern matching |
| `DateTime.now()` in business logic | Not testable | Inject a `Clock` or `DateTimeProvider` |

---

## Crashlytics Non-Fatal Reporting Standards

### When to record

Record a non-fatal error when:
- A caught exception degrades the user experience (silent fallback is NOT acceptable)
- An API call fails after all retries are exhausted
- A sync operation fails
- An expected-but-rare state is reached (empty required field, unexpected server response shape)

### When NOT to record

Do NOT record a non-fatal for:
- Expected network timeouts on poor connections: log as an analytics event instead
- User-cancelled operations
- Expected empty states (no search results, empty feed)

### Implementation pattern

```dart
// lib/core/crash/record_non_fatal.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';

// Pattern: wrap non-fatal reporting with debug guard and typed context
void recordNonFatal(
  Object error,
  StackTrace stack, {
  Map<String, String>? context,
}) {
  if (kDebugMode) {
    debugPrint('Non-fatal: $error\n$stack');
    return;
  }
  context?.forEach(FirebaseCrashlytics.instance.setCustomKey);
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: false);
}
```

Place this function in `core/analytics/crashlytics_reporter.dart`. All non-fatal recording in the app goes through this function: never call `FirebaseCrashlytics.instance.recordError` directly from feature code.

---

## Testing: Streaming & Async Provider Coverage

Every Riverpod `StreamProvider` or `AsyncNotifier` that wraps a gRPC or Realtime stream must have:

1. A unit test verifying the provider emits data on stream events
2. A unit test verifying the provider transitions to error state on stream error
3. A unit test verifying the subscription is cancelled on provider dispose (no exception thrown)

```dart
// test/features/chat/chat_messages_provider_test.dart
import 'dart:async';

import 'package:app/features/chat/chat_messages_provider.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';

class MockChatDataSource implements ChatDataSource {
  MockChatDataSource({required this.stream});

  @override
  final Stream<ChatMessage> stream;
}

// Pattern: test StreamProvider with a broadcast StreamController
void main() {
  const tMessage = ChatMessage('hello');

  test('chat provider emits messages from gRPC stream', () async {
    final controller = StreamController<ChatMessage>.broadcast();
    final container = ProviderContainer(
      overrides: [
        chatDataSourceProvider.overrideWith(
          (ref) => MockChatDataSource(stream: controller.stream),
        ),
      ],
    );
    addTearDown(container.dispose);
    final messages = container.listen(chatMessagesProvider, (_, _) {});

    // Initial state is loading
    expect(messages.read().isLoading, true);

    // Emitting data transitions to AsyncData
    controller.add(tMessage);
    await Future<void>.delayed(Duration.zero);
    expect(messages.read().value, [tMessage]);

    // Emitting error transitions to AsyncError
    controller.addError(Exception('stream error'));
    await Future<void>.delayed(Duration.zero);
    expect(messages.read().hasError, true);

    // Dispose does not throw
    await controller.close();
  });
}
```

---

## Golden Tests for Design System Components

Every design system component in `shared/ui/` with 2+ visual states requires a golden test covering each state.

```dart
// test/shared/ui/glass_card_test.dart
import 'package:app/core/theme/app_theme.dart';
import 'package:app/shared/ui/glass_card.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

// Pattern: golden test with explicit theme and deterministic rendering
void main() {
  testWidgets('GlassCard golden — light mode', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        theme: AppTheme.light(),
        home: const Scaffold(
          body: Center(child: GlassCard(child: Text('Test'))),
        ),
      ),
    );
    await expectLater(
      find.byType(GlassCard),
      matchesGoldenFile('goldens/glass_card_light.png'),
    );
  });

  testWidgets('GlassCard golden — dark mode', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        theme: AppTheme.dark(),
        home: const Scaffold(
          body: Center(child: GlassCard(child: Text('Test'))),
        ),
      ),
    );
    await expectLater(
      find.byType(GlassCard),
      matchesGoldenFile('goldens/glass_card_dark.png'),
    );
  });
}
```

**Update goldens intentionally** (never as a side effect of CI failure):

```bash
flutter test --update-goldens test/widget/golden/
```

**Rule:** Golden files are committed to git. They are the visual contract for the design system. A PR that unexpectedly changes golden files must be reviewed for unintentional visual regressions.

**Tag golden tests** with `@Tags(['golden'])` to allow selective exclusion from standard test runs and explicit inclusion in CI golden check step.

---

## Updated CI Quality Gate

Full pipeline including Firebase credential injection and golden test step:

```yaml
# .github/workflows/flutter_quality.yml
name: Flutter Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.47.x'
          channel: 'stable'
          cache: true

      - name: Inject Firebase credentials
        run: |
          echo $GOOGLE_SERVICES_JSON | base64 -d > android/app/google-services.json
          echo $GOOGLE_SERVICE_INFO_PLIST | base64 -d > ios/Runner/GoogleService-Info.plist
        env:
          GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
          GOOGLE_SERVICE_INFO_PLIST: ${{ secrets.GOOGLE_SERVICE_INFO_PLIST }}

      - name: Install dependencies
        run: flutter pub get

      - name: Run code generation
        run: dart run build_runner build

      - name: Analyze
        run: flutter analyze --fatal-infos --fatal-warnings

      - name: Test with coverage
        run: flutter test --coverage --exclude-tags=golden

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep 'lines' | awk '{print $2}' | tr -d '%')
          if (( $(echo "$COVERAGE < 75" | bc -l) )); then
            echo "Coverage $COVERAGE% below 75% threshold"
            exit 1
          fi

      - name: Golden test check
        run: flutter test test/widget/golden/ --tags=golden
```
