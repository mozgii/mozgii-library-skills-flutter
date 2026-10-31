---
name: dart-primary-constructors
description: Writes and migrates to Dart primary constructors - declaring parameters in the class header, the in-body this part, const headers, empty-body semicolons, and the short new and factory constructor forms. Use when asked to shorten a class, use a primary constructor, migrate constructors to the new syntax, or fix errors such as non_redirecting_generative_constructor_with_primary.
metadata:
  tags:
  - dart
  - language
  - constructors
---
# Dart Primary Constructors

A primary constructor puts a class's parameters in its header, so a small value class needs one line rather than a field list plus a constructor. It trims the boilerplate that most model and widget classes repeat. The feature is on by default from Dart 3.13.

## Step 0: Check the Language Version

```bash
dart --version
grep -n "sdk:" pubspec.yaml
```

| Dart | Status |
|---|---|
| 3.13 and later | Enabled, provided the package's lower SDK bound is `^3.13.0` or higher |
| 3.12 | Experimental: pass `--enable-experiment=primary-constructors`, or list it in `analysis_options.yaml` |
| 3.11 and earlier | Not available |

```yaml
# analysis_options.yaml, on Dart 3.12 only
analyzer:
  enable-experiment:
    - primary-constructors
```

A package that still supports older SDKs has to keep the old syntax, because raising the lower bound drops those users.

## When to Migrate

| Class | Use a primary constructor | Why |
|---|---|---|
| A small value class or DTO | Yes | Nothing is left but the header |
| A widget with a few `final` fields | Yes | `super.key` fits in the header |
| A class with heavy validation | Often | An in-body `this :` part keeps it readable |
| Several generative constructors | Only if they redirect | Other generative constructors must redirect to the primary |
| An extension type | Always | It has always required one |
| Public API on a package with old SDK users | No | The lower bound would have to rise |

## The Three Kinds of Parameter

| Written as | Kind | Effect |
|---|---|---|
| `final int x` or `var int x` | Declaring | Creates a field and initializes it |
| `this.x`, `super.x` | Initializing | Sets an existing field, or forwards to the superclass |
| `int y` | Plain | Exists during initialization only; no field |

## Worked Example: An Expense Model and a Widget

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
import 'package:material_ui/material_ui.dart';

// Before
class MoneyOld {
  const MoneyOld(this.cents, this.currency);

  final int cents;
  final String currency;
}

// After: `const` goes before the class name, and an empty body becomes a semicolon
class const Money(final int cents, final String currency);

// Validation and a derived field move to the in-body part
class Expense(final String note, final int cents, int taxPercent) {
  final int taxCents;

  this : assert(cents >= 0), taxCents = cents * taxPercent ~/ 100;

  int get totalCents => cents + taxCents;
}

// Named parameters, a default and super.key work as usual
class ExpenseTile({
  super.key,
  required final String note,
  final bool highlighted = false,
}) extends StatelessWidget {
  @override
  Widget build(BuildContext context) => ListTile(
        title: Text(note),
        selected: highlighted,
      );
}

// A plain parameter forwarded to the superclass in the in-body part
class TaxedExpense(final String region, int cents) extends Base {
  this : super(cents);
}

class Base(final int cents);

// Enums and extension types take the same header
enum const Currency(final String symbol) {
  usd(r'$'),
  eur('€');
}

extension type const ExpenseId(String value);
```

## The Other Syntax Changes

An empty body can be a semicolon on classes, mixin classes, mixins, extensions and extension types: `mixin Audited;`, `extension Helpers on Money;`.

A constructor in the body can drop the class name:

| Long form | Short form |
|---|---|
| `Money.zero() : ...` | `new zero() : ...` |
| `const Money.zero();` | `const new zero();` |
| `Money() { }` | `new() { }` |
| `factory Money() => ...` | `factory() => ...` |
| `factory Money.parse(String s) => ...` | `factory parse(String s) => ...` |

The body of the primary constructor itself goes in `this { ... }`. An extension type's representation parameter can't use `var`; `final` is optional and implied.

## Verify

```bash
dart analyze
dart fix --dry-run       # most diagnostics here come with a quick fix
dart test
```

Migrate one class, then run the analyzer: code that used the old constructor keeps working because the constructor's name and parameters are unchanged. If you moved fields, check `toString` and equality still see them.

## Gotchas

1. **`non_redirecting_generative_constructor_with_primary`.** Symptom: `Classes with primary constructors can't have non-redirecting generative constructors` on an in-body constructor. Cause: with a primary constructor, other generative constructors must redirect. Fix: use `new named() : this(0, 'USD');`, or a `factory`.
2. **`field_initialized_in_declaration_and_initializer_of_primary_constructor`.** Symptom: `Fields can't be initialized in both the primary constructor and at their declaration`. Cause: it's initialized in its declaration and in `this :`. Fix: remove one.
3. **A `late` field can't read a parameter.** Symptom: `Undefined name` on the parameter in its initializer. Cause: `late` initializers run after construction, when parameters are gone. Fix: make the field non-late, or read a declaring parameter's field.
4. **`representation_field_modifier`.** Symptom: an error on an extension type. Cause: `var` on the representation parameter. Fix: remove it.
5. **`experiment_not_enabled_off_by_default` on a correct class.** Symptom: `This requires the 'primary-constructors' language feature to be enabled` on the header. Cause: the package's lower SDK bound is below 3.13. Fix: raise it, or keep the old syntax.
6. **Freezed rejects `final` in a factory.** Symptom: an error from `const factory User({required final String id})`. Cause: Dart 3.13 no longer allows `final` there, and Freezed 4 follows. Fix: drop `final`.

## Anti-Patterns

- Migrating a public package's classes without checking its SDK constraint.
- Forcing a long, documented field list into the header. Field docs read better in the body.
- Keeping several generative constructors and fighting the redirect rule. Use factories.

## Pre-Merge Checklist

- [ ] The package's lower SDK bound allows the syntax
- [ ] `const` sits before the class name where the old constructor was const
- [ ] `dart analyze` and `dart test` pass
- [ ] Field docs survived the move

## See Also

- `dart-patterns` for object patterns over these classes
- `flutter-core` for Freezed models
- `dart-analyze-and-fix` for `dart fix`
