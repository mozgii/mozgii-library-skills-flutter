---
name: dart-build-cli-tool
description: Builds Dart command-line tools - project tasks, code generators and release scripts - with package:args and CommandRunner, clean stdout and stderr, POSIX exit codes, signal handling, finding the Dart SDK from a compiled binary, and tests with test_process. Use when asked to write a Dart CLI, add a tool/ script to a project, parse arguments, compile with dart compile exe, or fix exit codes.
metadata:
  tags:
  - dart
  - cli
  - tooling
---
# Dart Command-Line Tools

Flutter projects collect scripts: generators, asset checks, release helpers. Written in Dart, they share the app's language and packages. A good tool behaves well in a pipeline: results go to stdout, problems go to stderr, and the exit code tells CI what happened.

## Step 0: Set Up

```bash
dart --version
dart create -t cli tool_expense_report
dart pub add args cli_util io
```

For a single script inside an app, `tool/check_assets.dart` run with `dart run tool/check_assets.dart` is enough. Declare `executables:` in `pubspec.yaml` when others should run it by name.

## Choosing a Structure

| Tool | Use | Why the others lose here |
|---|---|---|
| One job, a few flags | `ArgParser` | `CommandRunner` is more than you need |
| Subcommands, like `git` | `CommandRunner<int>` with `Command<int>` classes | A hand-written switch has no help output |
| Many options, also from a config file | `package:build_cli` for generated option classes | Manual merging gets long |

## Output and Exit Codes

Write results to stdout and everything else to stderr, so `tool | jq` keeps working. Prefer `stdout.writeln` to `print`, which trips the `avoid_print` lint. With `--json`, send data to stdout and logs to stderr. Colour only when the terminal supports it:

```dart
import 'dart:io';

bool get useAnsi =>
    stdout.hasTerminal &&
    stdout.supportsAnsiEscapes &&
    !Platform.environment.containsKey('NO_COLOR');
```

| Code | Means | `package:io` name |
|---|---|---|
| 0 | Success | `ExitCode.success` |
| 1 | General failure | |
| 64 | Bad usage: wrong flags or arguments | `ExitCode.usage` |
| 65 | Bad input data | `ExitCode.data` |
| 70 | Internal error | `ExitCode.software` |
| 130 | Interrupted by Ctrl-C (128 + 2) | |

A usage error prints the message and the usage text to stderr, and exits 64. Usage goes to stdout only when someone asks with `--help`. A missing `mandatory: true` option throws when you read it, so catch that too.

## Worked Example: An Expense Report Tool

`CommandRunner<int>` returns the exit code from each command. `main` stays small: it sets `exitCode` and lets the process end on its own, which flushes stdout. Calling `exit()` directly can lose buffered output.

```dart
// bin/report.dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'package:args/command_runner.dart';
import 'package:io/io.dart';

Future<void> main(List<String> arguments) async {
  final runner = CommandRunner<int>('report', 'Expense reports.')
    ..addCommand(SummaryCommand());

  // Ctrl-C ends with 130, the POSIX code for SIGINT.
  final interrupted = ProcessSignal.sigint.watch().listen((_) {
    stderr.writeln('Interrupted');
    exit(130);
  });

  try {
    exitCode = await runner.run(arguments) ?? ExitCode.success.code;
  } on UsageException catch (e) {
    stderr
      ..writeln(e.message)
      ..writeln(e.usage);
    exitCode = ExitCode.usage.code;
  } on ArgumentError catch (e) {
    stderr
      ..writeln(e.message)
      ..writeln(runner.usage);
    exitCode = ExitCode.usage.code;
  } finally {
    await interrupted.cancel();
  }
}

class SummaryCommand extends Command<int> {
  SummaryCommand() {
    argParser
      ..addOption('input', abbr: 'i', mandatory: true, help: 'CSV of expenses.')
      ..addFlag('json', help: 'Print JSON for machines.')
      ..addMultiOption('category', abbr: 'c', help: 'Limit to these categories.');
  }

  @override
  String get name => 'summary';

  @override
  String get description => 'Sums expenses per category.';

  @override
  Future<int> run() async {
    final results = argResults!;
    final file = File(results.option('input')!);
    if (!file.existsSync()) {
      stderr.writeln('No such file: ${file.path}');
      return ExitCode.noInput.code;
    }

    final only = results.multiOption('category').toSet();
    final totals = <String, int>{};
    await for (final line in file.openRead().transform(utf8.decoder).transform(const LineSplitter())) {
      final [category, cents] = line.split(',');
      if (only.isEmpty || only.contains(category)) {
        totals[category] = (totals[category] ?? 0) + int.parse(cents);
      }
    }

    if (results.flag('json')) {
      stdout.writeln(jsonEncode(totals));
    } else {
      totals.forEach((category, cents) => stdout.writeln('$category\t${cents / 100}'));
    }
    return ExitCode.success.code;
  }
}
```

`results.flag`, `.option` and `.multiOption` (from `args` 2.5) return typed values, so there are no casts.

## Spawning Dart From a Compiled Tool

Once compiled, `Platform.resolvedExecutable` is your own binary, not `dart`. Running it as `dart` calls your tool again, which loops or fails on unknown flags. Since `cli_util` 0.6.0, `sdkPath` is nullable, and `dartExecutable` gives the binary itself:

```dart
import 'dart:io';

import 'package:cli_util/cli_util.dart';
import 'package:io/io.dart';

Future<int> formatTree(String dir) async {
  final dart = dartExecutable; // null when no SDK can be found
  if (dart == null) {
    stderr.writeln('No Dart SDK found: set DART_SDK or put dart on PATH.');
    return ExitCode.unavailable.code;
  }
  final result = await Process.run(dart, ['format', dir]);
  stderr.write(result.stderr);
  return result.exitCode;
}
```

Pass arguments as a list. Don't build a shell string, because a file name with a space or a `;` would run as a command.

## Compiling and Distributing

```bash
dart compile exe bin/report.dart -o build/report   # a standalone native binary
dart pub global activate --source path .           # try the executable by name
```

A compiled binary targets the OS and architecture it was built on, so build one per platform in CI.

## Verify

Test in two layers: command logic directly, and the real binary with `test_process` and `test_descriptor`.

```bash
dart pub add dev:test dev:test_process dev:test_descriptor
```

```dart
import 'package:test/test.dart';
import 'package:test_descriptor/test_descriptor.dart' as d;
import 'package:test_process/test_process.dart';

void main() {
  test('sums a category and exits 0', () async {
    await d.file('in.csv', 'food,100\nfood,250\n').create();

    final process = await TestProcess.start(
      'dart',
      ['run', 'bin/report.dart', 'summary', '--input', d.path('in.csv'), '--json'],
    );

    await expectLater(process.stdout, emits('{"food":350}'));
    await process.shouldExit(0);
  });

  test('a missing option exits 64 and writes to stderr', () async {
    final process = await TestProcess.start('dart', ['run', 'bin/report.dart', 'summary']);
    await process.shouldExit(64);
  });
}
```

```bash
dart analyze && dart test
dart run bin/report.dart summary; echo "exit: $?"     # expect 64
```

## Gotchas

1. **The output is cut off in a pipe.** Symptom: the last lines are missing. Cause: `exit()` ran before stdout flushed. Fix: set `exitCode` and return.
2. **Infinite self-invocation.** Symptom: a compiled tool spawns itself. Cause: `Platform.resolvedExecutable` used as `dart`. Fix: use `cli_util`'s `dartExecutable`, and handle null.
3. **CI passes a failed run.** Symptom: green with errors in the log. Cause: the tool printed an error but exited 0. Fix: return a non-zero code.
4. **Colour codes appear in a log file.** Symptom: `[32m` in the output. Cause: no terminal check. Fix: check `hasTerminal`, and respect `NO_COLOR`.
5. **Ctrl-C leaves temp files.** Symptom: litter after an interrupt. Cause: the signal skipped cleanup. Fix: clean up in the `sigint` handler.
6. **`SIGTERM` errors on Windows.** Symptom: `SignalException`. Cause: Windows lacks most signals. Fix: watch only `sigint` there.

## Anti-Patterns

- Using `print` for errors. It goes to stdout and corrupts piped data.
- Using `results['input'] as String`. Use the typed accessors.
- Running `Process.run('sh', ['-c', ...])` with interpolated input.
- Putting all logic in `main`. Commands should be classes you can test.

## Pre-Merge Checklist

- [ ] Data goes to stdout, and diagnostics to stderr
- [ ] Bad usage exits 64 with usage text
- [ ] Every failure exits non-zero
- [ ] `--help` works on each command
- [ ] Tests assert on exit codes

## See Also

- `dart-path-handling` for file arguments
- `dart-unit-test` for command logic
- `dart-analyze-and-fix` for `avoid_print`
