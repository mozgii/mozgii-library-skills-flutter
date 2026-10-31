---
name: flutter-implement-json-serialization
description: Turns JSON into typed Dart models and back with dart:convert, choosing between hand-written fromJson and toJson, pattern matching, json_serializable and Freezed, and parsing large payloads off the UI thread with compute or Isolate.run. Use when asked to parse an API response, write fromJson, map JSON keys to a class, fix a type cast error from JSON, or stop JSON parsing from dropping frames.
metadata:
  tags:
  - flutter
  - json
  - models
---
# JSON Serialization in Flutter

JSON arrives as untyped maps, and every wrong assumption about its shape becomes a runtime crash. A typed model with one checked entry point keeps those assumptions in one place, so a server change produces a clear `FormatException` at the boundary rather than a `null` deep in the UI.

## Step 0: See What the Project Uses

```bash
flutter --version
grep -nE "^\s+(json_annotation|json_serializable|freezed_annotation|built_value):" pubspec.yaml
```

If the project already generates models, follow it. Hand-written and generated models can live together, but one style per folder keeps reviews simple. The examples also import `http: ^1.6.0`.

## Picking an Approach

| Situation | Choose | Why the others lose here |
|---|---|---|
| A few small models, no build step wanted | Hand-written with `dart:convert` | Code generation adds a build step for little gain |
| Untrusted shape, you want validation in one expression | A switch with map patterns | Plain casts throw a `TypeError` with no context |
| Dozens of models, or nested ones | `json_serializable` | Hand-written mappers drift from the fields |
| Models that also need `copyWith` and equality | Freezed with `json_serializable` (`flutter-core`) | You'd hand-write three things instead of one |
| Payloads of thousands of objects | Any of the above, inside `Isolate.run` | Parsing on the UI isolate drops frames |

`dart:mirrors` isn't available in Flutter, so there's no reflection-based mapper. Every approach above is either written by hand or generated at build time.

## The Basic Calls

`jsonDecode` returns `dynamic`, usually a `Map<String, dynamic>` or a `List<dynamic>`. Cast it right away so the rest of the code is typed. `jsonEncode` calls `toJson()` on your objects when you pass them inside a map or a list.

```dart
// lib/data/json_codec.dart
import 'dart:convert';

import 'models/expense_dto.dart';

Map<String, dynamic> decodeExpense(String body) =>
    jsonDecode(body) as Map<String, dynamic>;

String encodeExpense(ExpenseDto expense) =>
    jsonEncode({'expense': expense}); // calls expense.toJson()
```

## Worked Example: A Checked Expense Model

The factory validates every required key in one pattern and keeps optional fields apart. A map pattern checks that a key exists, so `{'note': String? note}` fails when the key is missing, even though the type allows null. Read optional keys from the matched map instead.

```dart
// lib/data/models/expense_dto.dart
class ExpenseDto {
  const ExpenseDto({
    required this.id,
    required this.amountCents,
    required this.spentAt,
    this.note,
  });

  factory ExpenseDto.fromJson(Map<String, dynamic> json) => switch (json) {
        {
          'id': String id,
          'amount_cents': int amountCents,
          'spent_at': String spentAt,
        } =>
          ExpenseDto(
            id: id,
            amountCents: amountCents,
            spentAt: DateTime.parse(spentAt),
            note: json['note'] as String?, // optional: may be absent
          ),
        _ => throw FormatException('Not an expense: ${json.keys.toList()}'),
      };

  final String id;
  final int amountCents;
  final DateTime spentAt;
  final String? note;

  Map<String, dynamic> toJson() => {
        'id': id,
        'amount_cents': amountCents,
        'spent_at': spentAt.toUtc().toIso8601String(),
        if (note case final value?) 'note': value,
      };
}
```

Money is an `int` of cents here, because a `double` can't hold most decimal fractions exactly. JSON numbers can also arrive as `int` or `double`: when a field might be either, match `num amount` and convert with `.toDouble()`.

Nested objects call their own factory, and lists map over their items:

```dart
// lib/data/models/line_item.dart
class LineItem {
  const LineItem(this.sku);

  factory LineItem.fromJson(Map<String, dynamic> json) =>
      LineItem(json['sku'] as String);

  final String sku;
}

List<LineItem> lineItemsOf(Map<String, dynamic> json) =>
    (json['items'] as List<dynamic>)
        .cast<Map<String, dynamic>>()
        .map(LineItem.fromJson)
        .toList();
```

## Parsing in the Background

Parsing takes place on the UI isolate unless you move it. Anything that takes longer than a frame, about 16 ms on a 60 Hz screen, shows as jank. A single object is fine on the main isolate. A list of thousands isn't.

```dart
// lib/data/expense_api.dart
import 'dart:convert';
import 'dart:isolate';

import 'package:http/http.dart' as http;

import 'models/expense_dto.dart';

Future<List<ExpenseDto>> fetchExpenses(http.Client client, Uri url) async {
  final response = await client.get(url);
  if (response.statusCode != 200) {
    throw http.ClientException('Expenses failed: ${response.statusCode}', url);
  }
  final body = response.body;
  return Isolate.run(() {
    final rows = (jsonDecode(body) as List<dynamic>).cast<Map<String, dynamic>>();
    return rows.map(ExpenseDto.fromJson).toList();
  });
}
```

`compute(parseExpenses, body)` from `package:flutter/foundation.dart` does the same job and needs a top-level or static function. On web neither runs in a real isolate: `compute` runs the work on the main thread, and `Isolate.run` isn't supported there, so use `compute` for code that must also run on web.

## Generated Models With json_serializable

```yaml
dependencies:
  json_annotation: ^4.12.0
dev_dependencies:
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
  json_serializable: ^6.15.2
```

```dart
// lib/data/models/budget_dto.dart
import 'package:json_annotation/json_annotation.dart';

part 'budget_dto.g.dart';

@JsonSerializable(fieldRename: FieldRename.snake)
class BudgetDto {
  const BudgetDto({required this.categoryId, required this.limitCents});

  factory BudgetDto.fromJson(Map<String, dynamic> json) => _$BudgetDtoFromJson(json);

  final String categoryId;
  final int limitCents;

  Map<String, dynamic> toJson() => _$BudgetDtoToJson(this);
}
```

```bash
dart run build_runner build
```

## Verify

Test both directions, and test the failure. A round trip catches a key that only one side renamed.

```dart
import 'package:expense_tracker/data/models/expense_dto.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  const valid = {'id': 'e1', 'amount_cents': 1250, 'spent_at': '2026-10-30T09:00:00Z'};

  test('reads required fields and tolerates a missing note', () {
    final dto = ExpenseDto.fromJson(valid);
    expect(dto.amountCents, 1250);
    expect(dto.note, isNull);
  });

  test('survives a round trip', () {
    final dto = ExpenseDto.fromJson({...valid, 'note': 'Lunch'});
    expect(ExpenseDto.fromJson(dto.toJson()).note, 'Lunch');
  });

  test('rejects a wrong type with a FormatException', () {
    expect(
      () => ExpenseDto.fromJson({...valid, 'amount_cents': '1250'}),
      throwsFormatException,
    );
  });
}
```

```bash
flutter test test/data/models/
```

## Gotchas

1. **`type 'int' is not a subtype of type 'double'`.** Symptom: a crash only for round numbers. Cause: JSON's `10` decodes as `int`. Fix: read it as `num`, then call `.toDouble()`.
2. **`type 'List<dynamic>' is not a subtype of type 'List<String>'`.** Symptom: a crash on a list field. Cause: a plain cast on a decoded list. Fix: `.cast<String>()`, or map each item.
3. **`Null check operator used on a null value` far from the parser.** Symptom: a crash in a widget. Cause: a missing key was accepted. Fix: validate at the boundary with a pattern.
4. **An optional key fails a map pattern.** Symptom: `FormatException` for a valid response that omits the key. Cause: map patterns require the key to exist. Fix: read optional keys from the map, as the `note` field does.
5. **`_$BudgetDtoFromJson isn't defined`.** Symptom: an analyzer error after adding a model. Cause: the generator hasn't run, or the `part` line is missing. Fix: add `part`, and run `build_runner`.
6. **Dates shift by hours.** Symptom: an expense appears on the wrong day. Cause: a local time was sent without an offset. Fix: send `toUtc().toIso8601String()`, and convert for display.

## Anti-Patterns

- Passing `Map<String, dynamic>` through the app. Convert at the edge.
- Returning `null` from a parser on failure. Throw, so the caller's error handling runs.
- Decoding the same string twice. Decode once, then pass the map or the model.
- Using a `double` for money. Use integer cents.

## Pre-Merge Checklist

- [ ] Every model has a single `fromJson` entry point
- [ ] Required fields fail loudly, and optional ones tolerate a missing key
- [ ] Large lists parse off the UI isolate
- [ ] Tests cover a round trip and a malformed payload
- [ ] Dates are sent in UTC

## See Also

- `flutter-use-http-package` for fetching the JSON
- `dart-patterns` for map and list patterns
- `flutter-core` for Freezed models
