---
name: dart-analyze-and-fix
description: Finds and fixes Dart errors - analysis_options.yaml with strict type checks, dart analyze, dart fix, dart format, suppressing a false positive properly, and resolving null safety, type and override errors, plus runtime failures such as TypeError and LateInitializationError. Use when asked to fix analyzer errors, clean up lints, set up strict analysis, apply dart fix, or debug a Dart exception from a stack trace.
metadata:
  tags:
  - dart
  - analysis
  - lints
  - debugging
---
# Dart Analysis and Error Fixing

The analyzer catches most bugs before anything runs, provided it's told to be strict. With strict checks on, implicit `dynamic` turns into a build error rather than a crash in the field. This skill sets that up, then walks through fixing what the analyzer and the runtime report.

## Step 0: See the Current State

```bash
dart --version
ls analysis_options.yaml
dart analyze --fatal-infos 2>&1 | tail -3
```

Flutter projects can use `flutter analyze`. Through the Dart MCP server, the same jobs are `analyze_files`, `dart_fix` and `dart_format` (`flutter-agent-tooling`).

## Which Tool for Which Problem

| Problem | Use | Why the others lose here |
|---|---|---|
| Many mechanical lints or a deprecated API | `dart fix --apply` | By hand it's slow and error-prone |
| Style and line breaks | `dart format .` | Nobody should argue about spacing |
| A real type or null error | A manual fix | `dart fix` won't change logic |
| A false positive on one line | `// ignore: code` with a reason | Disabling the rule hides real cases |
| Generated code | `analyzer: exclude:` | Editing it is pointless |
| An exception at runtime | The stack trace, then `get_runtime_errors` with an agent | The analyzer can't see runtime values |

## A Strict analysis_options.yaml

```yaml
include: package:flutter_lints/flutter.yaml   # pure Dart: package:lints/recommended.yaml

analyzer:
  language:
    strict-casts: true       # no implicit downcast from dynamic
    strict-inference: true   # inference never falls back to dynamic silently
    strict-raw-types: true   # List needs its type argument
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
  errors:
    todo: ignore
    unawaited_futures: warning

linter:
  rules:
    avoid_print: true
    unawaited_futures: true
    use_super_parameters: true
    avoid_catching_errors: true

formatter:
  page_width: 100
  trailing_commas: preserve
```

Under `linter: rules:`, use either a map (`rule: true`) to override an included set, or a list (`- rule`) for a fresh one. Don't mix the two. `very_good_analysis` is a stricter ready-made set (`flutter-quality`). Plugins go under `analyzer: plugins:`, and their package goes in `dev_dependencies`.

## The Fix Loop

```bash
dart analyze .
dart fix --dry-run            # preview
dart fix --apply
dart format .
dart analyze . --fatal-infos
dart test
```

Read the dry run: `dart fix` only offers fixes for rules that are on, so enable a lint first if you want its fix. `--fatal-infos` makes info-level issues fail, which suits CI.

## Worked Example: Three Errors From a Report Feature

**A list inferred as dynamic.** The list literal has no element type, so it can't be passed on as `List<int>`.

```dart
void printReport(List<int> totals) {}

void collect() {
  final totals = [];
  totals.add(1200);
  // Error: The argument type 'List<dynamic>' can't be assigned to 'List<int>'
  printReport(totals);
}
```

```dart
void printReport(List<int> totals) {}

void collect() {
  final totals = <int>[];
  totals.add(1200);
  printReport(totals);
}
```

**A nullable read.** Decide first whether null is a real state here.

```dart
class Expense {
  const Expense(this.note);
  final String? note;
}

// Error: The property 'length' can't be unconditionally accessed
int noteLength(Expense e) => e.note.length;
```

```dart
class Expense {
  const Expense(this.note);
  final String? note;
}

int noteLength(Expense e) => e.note?.length ?? 0;   // null is a valid state
```

Use `late` only when initialization is truly guaranteed before use, because it swaps a compile error for a possible `LateInitializationError`. A local variable can usually be promoted with an `if (x != null)` instead. A field can't, so copy it to a local first.

**An override that narrows a parameter.** A subclass can't accept less than its parent, because callers holding the parent type could pass anything.

```dart
class Expense {}

class Report { void add(Object row) {} }

class ExpenseReport extends Report {
  // Error: 'ExpenseReport.add' isn't a valid override of 'Report.add'
  @override
  void add(Expense row) {}
}
```

Widen the parameter back to `Object`, or mark it `covariant` if the narrower type is truly part of the design:

```dart
class Expense {}

class Report { void add(Object row) {} }

class ExpenseReport extends Report {
  @override
  void add(covariant Expense row) {}
}
```

## Suppressing a Diagnostic Properly

Add a reason, so the next reader can remove it when it stops being true.

| Scope | How |
|---|---|
| One line | `// ignore: unused_element` above the line, or at its end |
| One file | `// ignore_for_file: unused_element` at the top |
| All lints in a file | `// ignore_for_file: type=lint` |
| A plugin's diagnostic | `// ignore: some_plugin/some_code` |
| A line in `pubspec.yaml` | `# ignore: sort_pub_dependencies` above it |
| Whole paths | `analyzer: exclude:` globs |

## Reading Runtime Failures

Errors and exceptions mean different things in Dart. An `Exception` is a failure you expect and can recover from, so catch it. An `Error` such as `TypeError` or `ArgumentError` is a bug, so fix the code; `avoid_catching_errors` enforces that. Use `rethrow` to pass an exception on with its stack trace intact.

| Message | Cause | Fix |
|---|---|---|
| `type 'Null' is not a subtype of type 'String'` | A `!` or a cast on a null value, usually from JSON | Validate at the boundary |
| `LateInitializationError` | A `late` field was read before being set | Set it earlier, or make it nullable |
| `RangeError (index)` | An index past the end | Check `length`, or use a list pattern |
| `Bad state: No element` | `.first` on an empty list | Use `.firstOrNull` |
| `Concurrent modification during iteration` | The list changed in its own loop | Loop over a copy |
| `setState() called after dispose()` | Async work outlived the widget | Check `mounted` after each `await` |

With an agent, read the failure through `get_runtime_errors`, jump to the code with `lsp`, apply the fix, then `hot_reload` and read the errors again.

## Verify

```bash
dart analyze . --fatal-infos     # exit 0: nothing left
dart format --output=none --set-exit-if-changed .
dart test
```

`--set-exit-if-changed` makes the format check fail in CI without touching files.

## Gotchas

1. **`dart fix` found nothing, though the analyzer complains.** Symptom: the dry run is empty. Cause: no fix exists for that diagnostic, or the lint is off. Fix: enable the rule, or fix it by hand.
2. **Thousands of new errors.** Symptom: `strict-casts` lights up a whole project. Cause: years of implicit `dynamic`. Fix: enable one setting at a time, and start with the data layer.
3. **Errors inside `*.g.dart`.** Symptom: analyzer noise in generated code. Cause: stale output, or no exclude. Fix: re-run `build_runner`, and add the exclude.
4. **`The rule isn't defined`.** Symptom: a warning in `analysis_options.yaml`. Cause: the rule was removed or your SDK is too old. Fix: check dart.dev/lints.
5. **A cast compiles and then crashes.** Symptom: a `TypeError` at `as List<String>`. Cause: the object is really a `List<dynamic>`. Fix: use `.cast<String>()`, or map each element.
6. **A `field` won't promote.** Symptom: `can't be promoted because it's not a local variable`, or it's public. Cause: another getter could change it. Fix: copy it to a local, or use a pattern such as `if (user case final u?)`.

## Anti-Patterns

- Adding `!` until errors stop. Each one is a crash you've scheduled.
- Using `dynamic` to get past a type error. Find the real type.
- Catching `Error`, or an empty `catch`. The bug stays, and now it's silent.
- Disabling a rule for the whole project because of one line.

## Pre-Merge Checklist

- [ ] `dart analyze --fatal-infos` is clean
- [ ] `dart format` reports no changes
- [ ] Every ignore comment has a reason
- [ ] No new `dynamic`, no new `!` without a proof it can't be null
- [ ] Tests pass after `dart fix`

## See Also

- `flutter-agent-tooling` for the MCP tools
- `dart-manage-dependencies` for version errors
- `dart-unit-test` for pinning a fixed bug
