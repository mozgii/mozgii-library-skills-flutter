---
name: dart-patterns
description: Uses Dart 3 patterns well - switch expressions, sealed classes with exhaustiveness, records, map and list patterns for validating JSON and paths, guards, and when a plain if is better. Use when asked to use pattern matching, replace an if-else chain, model a sealed result type, destructure a record or a list, validate a map, or fix a non-exhaustive switch error.
metadata:
  tags:
  - dart
  - language
  - patterns
---
# Dart Patterns

Patterns let one expression check a value's shape and pull data out of it. Combined with `sealed` classes, the compiler proves every case is handled, so adding a state turns forgotten branches into build errors rather than bugs in production. They help most at boundaries, such as JSON and paths, and around state unions.

## Step 0: Check the Language Version

```bash
dart --version
grep -n "sdk:" pubspec.yaml
```

Patterns need a package's lower SDK bound at Dart 3.0 or above. Below that, the analyzer rejects them even on a newer SDK. The examples also import `test: ^1.33.0`.

## Picking the Construct

| You have | Use | Why the others lose here |
|---|---|---|
| A value to produce from several cases | Switch expression | A statement needs a mutable variable |
| Side effects per case | Switch statement | An expression requires a value in every arm |
| One shape to test, once | `if (x case Pattern)` | A switch with one arm is noise |
| A single type check on a local | `if (x is T)` | Promotion needs no new variable |
| A plain numeric comparison | `if (code >= 200 && code < 300)` | A relational pattern adds nothing alone |
| A family of states | `sealed` class with a switch | `abstract` gives no exhaustiveness |
| Several return values | A record and a destructuring `final` | A small class is more code |

## Pattern Syntax at a Glance

| Pattern | Example | Matches |
|---|---|---|
| Constant | `'admin'`, `404` | An equal value |
| Variable | `final name`, `String name` | Anything, binding it |
| Wildcard | `_` | Anything, discarding it |
| Relational | `>= 500`, `!= 0` | A comparison against a constant |
| Logical or | `'a' \|\| 'b'` | Either side; both must bind the same variables |
| Logical and | `> 0 && < 10` | Both sides; they can't bind the same name |
| Null check | `final s?` | A non-null value, promoted |
| Null assert | `final s!` | A non-null value, or it throws |
| Cast | `final n as int` | Any value, and it throws if the cast fails |
| List | `[a, b]`, `['api', ...final rest]` | Exact length, unless a rest element is present |
| Map | `{'id': int id}` | The key exists; extra keys are ignored |
| Record | `(x, y)`, `(:final name)` | The exact shape |
| Object | `Circle(:final radius)` | The type, then its getters |

## Worked Example: A Typed Payment Event

A server sends tagged JSON. The map pattern validates it, a sealed class models it, and the UI handles every case.

```dart
// lib/payment_event.dart
sealed class PaymentEvent {
  const PaymentEvent();

  factory PaymentEvent.fromJson(Map<String, dynamic> json) => switch (json) {
        {'type': 'paid', 'amount_cents': int cents} => Paid(cents),
        {'type': 'refunded', 'amount_cents': int cents, 'reason': String why} =>
          Refunded(cents, why),
        {'type': 'failed', 'code': int code} when code >= 500 => const Retryable(),
        {'type': 'failed', 'code': int code} => Declined(code),
        _ => throw FormatException('Unknown payment event: $json'),
      };
}

final class Paid extends PaymentEvent {
  const Paid(this.cents);
  final int cents;
}

final class Refunded extends PaymentEvent {
  const Refunded(this.cents, this.reason);
  final int cents;
  final String reason;
}

final class Retryable extends PaymentEvent {
  const Retryable();
}

final class Declined extends PaymentEvent {
  const Declined(this.code);
  final int code;
}

String describe(PaymentEvent event) => switch (event) {
      Paid(:final cents) => 'Paid ${cents / 100}',
      Refunded(:final cents, :final reason) => 'Refunded ${cents / 100}: $reason',
      Retryable() => 'Temporary problem, retrying',
      Declined(code: 402) => 'Card declined',
      Declined(:final code) => 'Payment failed ($code)',
    };
```

There's no default arm in `describe`, so adding a `Disputed` class makes the switch a compile error until someone handles it. A `_` arm would silence that protection, so leave it out for sealed types you own.

## Lists, Paths and Records

List patterns replace length checks and index arithmetic. They're at their best on URL segments and split strings.

```dart
String route(Uri uri) => switch (uri.pathSegments) {
      [] => 'home',
      ['expenses'] => 'list',
      ['expenses', final id] => 'detail $id',
      ['expenses', final id, 'receipts', ...final rest] when rest.length < 2 =>
        'receipts for $id',
      _ => 'not found',
    };

(int, String) parseLine(String line) {
  final [count, ...words] = line.split(' ');
  return (int.parse(count), words.join(' '));
}

void main() {
  final (qty, label) = parseLine('3 train tickets');
  print('$qty x $label');

  var (left, right) = ('a', 'b');
  (left, right) = (right, left); // swap with no temporary

  for (final MapEntry(:key, :value) in {'food': 12}.entries) {
    print('$key $value');
  }
}
```

The destructuring `final [count, ...words] = ...` throws a `StateError` when the list is empty, so it suits data you've already validated.

## Map Patterns and Missing Keys

A map pattern checks that the key exists, so `{'note': String? note}` doesn't match when `note` is absent, even though the type is nullable. Destructure required keys, then read optional ones from the map:

```dart
void saveNote(Map<String, Object?> json, void Function(String, String?) save) {
  if (json case {'id': String id}) {
    final note = json['note'] as String?;
    save(id, note);
  }
}
```

The analyzer can't check keys in a `Map<String, dynamic>`, so end such switches with `_ => throw FormatException(...)`. A silent `if case` skips malformed data.

## Verify

```bash
dart analyze
dart test
```

Add a subtype to a sealed class and confirm the analyzer flags each switch. For map patterns, test three payloads: valid, a missing optional key, and a wrong type.

```dart
// test/payment_event_test.dart
import 'package:app/payment_event.dart';
import 'package:test/test.dart';

void main() {
  test('missing optional key still parses', () {
    expect(describe(PaymentEvent.fromJson({'type': 'paid', 'amount_cents': 100})), contains('Paid'));
  });

  test('unknown type throws', () {
    expect(() => PaymentEvent.fromJson({'type': 'nope'}), throwsFormatException);
  });
}
```

## Gotchas

1. **`The type 'X' is not exhaustively matched`.** Symptom: an analyzer error on a switch expression. Cause: a sealed subtype has no arm. Fix: add one. For non-sealed types, add `_`.
2. **`This case is covered by the previous cases`.** Symptom: a dead-code warning. Cause: a broad pattern sits above a narrow one. Fix: put specific arms first.
3. **A valid payload hits the `_` arm.** Symptom: `FormatException` for good data. Cause: an optional key sits in the map pattern. Fix: read it from the map.
4. **`A variable must be bound in every branch`.** Symptom: an error on `||`. Cause: the two sides bind different names. Fix: bind the same names on both sides.
5. **`The variable is already defined`.** Symptom: a destructuring `final` shadows an outer variable. Cause: you meant to assign. Fix: use the assignment form, `(a, b) = record;`.
6. **Sealed classes can't be extended in another file.** Symptom: `can't be extended outside of its library`. Cause: that's what `sealed` does. Fix: keep the family in one library, using `part` if it's large.

## Anti-Patterns

- `if (x case final String s) use(s);` on a promotable local. Write `if (x is String) use(x);`.
- `switch (flag) { true => a, false => b }`. Use `flag ? a : b`.
- Using `if case` in a loop to drop malformed rows silently. Throw, so the bad data is seen.
- Destructuring one or two fields just to pass them on. Read the property.
- Reading `.first` and `.last` through `[first, ..., last]` on a list that can have one item. That pattern needs two or more.

## Pre-Merge Checklist

- [ ] Sealed families have no needless `_` arm
- [ ] JSON switches end in a throwing `_` arm
- [ ] Optional map keys are read outside the pattern
- [ ] Specific arms come before general ones
- [ ] `dart analyze` is clean

## See Also

- `flutter-implement-json-serialization` for model factories
- `dart-primary-constructors` for shorter class declarations
- `flutter-core` for sealed Freezed unions
