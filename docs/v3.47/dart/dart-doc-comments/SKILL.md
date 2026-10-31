---
name: dart-doc-comments
description: Writes Dart API documentation that dartdoc renders well - /// comments, first-sentence summaries, naming openers, reference links with square brackets, @docImport, and code samples kept compilable with the @example directive and regions. Use when asked to document a class or a package, write doc comments, add an example to the docs, or fix a comment_references warning.
metadata:
  tags:
  - dart
  - documentation
---
# Dart Documentation Comments

Doc comments appear in the IDE at every call site and on pub.dev, so they're the documentation people actually read. The conventions below are from Effective Dart, plus the `{@example}` directive, which keeps samples compiling rather than rotting inside a comment.

## Step 0: Check What's Enforced

```bash
dart --version
grep -nE "public_member_api_docs|comment_references" analysis_options.yaml
dart doc --dry-run 2>&1 | tail -3
```

Turn on `public_member_api_docs` for a published package, so a missing doc is a lint.

## What to Document

| Declaration | Doc comment? | Why |
|---|---|---|
| A public class, method, field or top-level function | Yes | It appears in the API reference |
| A library | Yes, on the `library;` directive | It's the package's front page |
| A private member | Only when it's subtle | dartdoc skips it |
| An `@override` that behaves the same | No | The parent's doc is inherited |
| A getter and setter pair | The getter only | Tooling warns if both have docs |

## The Rules

Use `///` for every doc comment, never `/** */`. Write full sentences. Make the first paragraph a single sentence that summarizes the element, then leave a `///` blank line, because dartdoc takes that sentence as the summary in lists. Doc comments go above annotations such as `@override`.

Pick the opening by the kind of element:

| Element | Opens with | Example |
|---|---|---|
| A method or function | A third-person verb | `/// Converts the amount to [currency].` |
| A non-boolean property | A noun phrase | `/// The total in cents.` |
| A boolean property | `Whether` | `/// Whether the expense was reimbursed.` |
| A class | A noun phrase | `/// A single entry in a monthly budget.` |
| A library | What it provides | `/// Budget tracking and currency conversion.` |

## Links and Formatting

Square brackets link any identifier in scope: `[Money]`, `[amount]`, `[Money.convert]`. Leave off parentheses, so `[String.contains]`, not `[String.contains()]`. Link an unnamed constructor as `[Money.new]`. Put keywords and literals in backticks: `` `null` ``, `` `true` ``. For a symbol the library doesn't import, add a `@docImport` on the `library;` directive instead of importing it for real.

Javadoc tags such as `@param`, `@return` and `@throws` don't exist in Dart. Weave parameters and failures into the prose. Name each code fence's language, because dartdoc treats an unlabeled block as Dart.

## Worked Example: A Documented Class

```dart
// lib/budget.dart
/// @docImport 'package:budget/src/rates.dart';
library;

/// An amount of money in one currency, held in minor units.
///
/// Values are immutable. Use [convert] to get the same amount in another
/// currency, and compare amounts only after converting them to one currency.
///
/// {@example /example/money.dart#convert}
class Money {
  /// Creates an amount of [cents] in [currency].
  ///
  /// Throws an [ArgumentError] if [currency] isn't a three-letter code.
  Money(this.cents, this.currency) {
    if (currency.length != 3) throw ArgumentError.value(currency, 'currency');
  }

  /// The amount in minor units, such as cents.
  final int cents;

  /// The ISO 4217 currency code.
  final String currency;

  /// Whether this amount is exactly zero.
  bool get isZero => cents == 0;

  /// Returns this amount in [target] at [rate], such as one from [RateTable].
  ///
  /// Rounds to the nearest minor unit. Returns this object unchanged when
  /// [target] matches [currency].
  Money convert(String target, double rate) =>
      target == currency ? this : Money((cents * rate).round(), target);
}
```

## Keeping Examples Compilable With @example

A sample inside a comment is never analyzed, so it breaks silently. Put it in a real file and pull it in.

```text
{@example <path>[#<region>] [lang=LANGUAGE] [indent=keep|strip]}
```

A path starting with `/` is from the package root, and anything else is relative to the current file. `lang` comes from the file extension unless you set it, and `indent` defaults to `strip`.

```dart
// example/money.dart
import 'package:budget/budget.dart';
import 'package:budget/src/rates.dart';

void main() {
  final rates = RateTable.fixed({'EUR': 1.1});

  // #region convert
  final lunch = Money(1200, 'EUR');
  final usd = lunch.convert('USD', rates.rate('EUR'));
  // #endregion convert

  print(usd.cents); // #hide
}
```

`#region name` and `#endregion name` mark what's shown, so a file can hold several regions. `#hide` drops a single line from the output, which keeps setup code compiling without showing it. Use a short inline fence only for a one-liner; anything longer belongs in an example file.

## Verify

```bash
dart analyze          # comment_references flags broken links
dart doc --dry-run    # finds problems without writing output
dart doc              # writes doc/api: open it and check the page
dart run example/money.dart
```

## Gotchas

1. **`The referenced name isn't visible in scope`.** Symptom: a `comment_references` warning. Cause: the target isn't imported. Fix: add `@docImport`, or use backticks.
2. **The summary runs on in the API list.** Symptom: a whole paragraph shows beside the class name. Cause: no blank `///` line after the first sentence. Fix: add it.
3. **A doc comment is ignored.** Symptom: nothing shows on hover. Cause: an empty line between the comment and the declaration, or `//` used by mistake. Fix: attach it directly, and use `///`.
4. **`Documentation on both getter and setter`.** Symptom: a dartdoc warning. Cause: both have comments. Fix: keep the getter's.
5. **`{@example}` renders nothing.** Symptom: a gap in the docs. Cause: a wrong path or a mistyped region. Fix: check for a leading `/`, and match the region name.
6. **An `[index]` link lands on the wrong thing.** Symptom: a bracket in plain text becomes a link. Cause: the text matched an identifier. Fix: use backticks.

## Anti-Patterns

- `/// Gets the amount.` on a getter. Use a noun phrase.
- `/// This method converts...`. Open with the verb.
- Restating the signature. Say what isn't obvious: units, failure modes, ownership.
- Copying the parent's doc onto an override.

## Pre-Merge Checklist

- [ ] Every public element has a one-sentence summary
- [ ] Failure modes are described in prose
- [ ] All bracket links resolve under `dart analyze`
- [ ] Samples longer than a line live in `example/`
- [ ] `dart doc --dry-run` is clean

## See Also

- `dart-analyze-and-fix` for lint setup
- `dart-primary-constructors` for class declarations
