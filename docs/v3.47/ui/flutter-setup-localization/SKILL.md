---
name: flutter-setup-localization
description: Adds localization to a Flutter app with gen-l10n - flutter_localizations, intl, ARB files, l10n.yaml, the generated AppLocalizations class, plurals, selects, date and currency formats, right-to-left layouts and tests. Use when asked to translate an app, add a language, set up i18n or l10n, fix a missing AppLocalizations import or package:flutter_gen error, or format dates and money per locale.
metadata:
  tags:
  - flutter
  - i18n
  - intl
---
# Flutter Localization

Flutter's localization tool turns ARB files into a typed class, so a missing translation key is a compile error rather than an empty label. It also covers the parts people forget: plurals, currencies, date order and right-to-left layouts. Set it up before strings spread through the widget tree, because every hard-coded string is a later edit.

## Step 0: Check Versions and Old Setups

```bash
flutter --version
grep -rn "flutter_gen/gen_l10n\|synthetic-package" lib l10n.yaml 2>/dev/null
grep -nE "generate:|flutter_localizations|intl:" pubspec.yaml
```

Any hit in the first search is out of date. Flutter stopped generating the synthetic `package:flutter_gen` in 3.32, and generated code now goes into your source tree.

### Upgrading From package:flutter_gen

1. Delete `synthetic-package` from `l10n.yaml`.
2. Make sure `flutter: generate: true` is in `pubspec.yaml`; it's required now.
3. Replace `package:flutter_gen/gen_l10n/app_localizations.dart` with the new path: `package:expense_tracker/l10n/app_localizations.dart`.
4. Run `flutter gen-l10n`.

## Choosing a Tool

| Situation | Choose | Why the others lose here |
|---|---|---|
| Most apps | `gen-l10n` with ARB files | Ships with Flutter, is typed, and translators know ARB |
| Translators work in another format | Third-party packages such as `slang` | ARB tooling would need a conversion step |
| Strings change without an app release | A remote strings service, with ARB as the fallback | Generated code is fixed at build time |

## Setup

```bash
flutter pub add flutter_localizations --sdk=flutter
flutter pub add intl:any
```

`intl: any` is deliberate: `flutter_localizations` pins the version of `intl` it needs.

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: any

flutter:
  generate: true
```

```yaml
# l10n.yaml, at the project root
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

`nullable-getter: false` makes `AppLocalizations.of(context)` non-null, which removes a `!` from every call. Generation runs on `flutter pub get` and `flutter run`, and `flutter gen-l10n` runs it alone. Files land in `arb-dir`, or `output-dir` when that's set.

### l10n.yaml Options Worth Knowing

| Key | Effect |
|---|---|
| `arb-dir` | Folder holding the ARB files; default `lib/l10n` |
| `output-dir` | Where generated Dart goes |
| `template-arb-file` | The source language file; default `app_en.arb` |
| `output-class` | The generated class name; default `AppLocalizations` |
| `untranslated-messages-file` | Writes a JSON report of missing strings |
| `preferred-supported-locales` | Puts a locale first in the list |
| `required-resource-attributes` | Requires an `@key` entry for every message |
| `use-deferred-loading` | Loads locales lazily on web |
| `use-escaping` | Lets single quotes escape braces |
| `relax-syntax` | Treats a stray brace as text |
| `use-named-parameters` | Makes generated methods use named parameters |

## Worked Example: A Localized Expense Summary

```json
{
  "@@locale": "en",
  "expensesTitle": "Expenses",
  "@expensesTitle": { "description": "Title of the expense list screen" },

  "welcomeBack": "Welcome back, {name}",
  "@welcomeBack": {
    "description": "Greeting on the home screen",
    "placeholders": { "name": { "type": "String", "example": "Ada" } }
  },

  "expenseCount": "{count, plural, =0{No expenses yet} =1{1 expense} other{{count} expenses}}",
  "@expenseCount": {
    "description": "Number of expenses this month",
    "placeholders": { "count": { "type": "int" } }
  },

  "monthlyTotal": "Total: {amount}",
  "@monthlyTotal": {
    "description": "Sum of this month's expenses",
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "simpleCurrency",
        "optionalParameters": { "decimalDigits": 2 }
      }
    }
  },

  "lastEntry": "Last entry on {date}",
  "@lastEntry": {
    "description": "Date of the newest expense",
    "placeholders": { "date": { "type": "DateTime", "format": "yMMMd" } }
  },

  "splitWith": "{relation, select, partner{Split with your partner} team{Split with your team} other{Split}}",
  "@splitWith": {
    "description": "Label on the split button",
    "placeholders": { "relation": { "type": "String" } }
  }
}
```

Add `lib/l10n/app_es.arb` with the same keys and no `@` entries. Plural languages differ, so a translator can add `few` and `many`, and `other` is the only required case.

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// lib/main.dart
import 'package:expense_tracker/l10n/app_localizations.dart';
import 'package:material_ui/material_ui.dart';

class ExpenseApp extends StatelessWidget {
  const ExpenseApp({super.key});

  @override
  Widget build(BuildContext context) => MaterialApp(
        // title can't use AppLocalizations here, since it sits above Localizations
        onGenerateTitle: (context) => AppLocalizations.of(context).expensesTitle,
        localizationsDelegates: const [
          AppLocalizations.delegate,
          ...GlobalMaterialLocalizations.delegates,
        ],
        supportedLocales: AppLocalizations.supportedLocales,
        home: const SummaryScreen(),
      );
}

class SummaryScreen extends StatelessWidget {
  const SummaryScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    return Scaffold(
      appBar: AppBar(title: Text(l10n.expensesTitle)),
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(l10n.welcomeBack('Ada')),
          Text(l10n.expenseCount(3)),
          Text(l10n.monthlyTotal(148.5)),
          Text(l10n.lastEntry(DateTime(2026, 10, 30))),
          Text(l10n.splitWith('team')),
        ],
      ),
    );
  }
}
```

`GlobalMaterialLocalizations.delegates` is the short form that also includes the Cupertino and Widgets delegates. On older setups you'll see the three listed one by one. With `material_ui`, the delegates come from that package (`flutter-core`, Step 0).

## Number and Date Formats

Number placeholders take a `format`. With 1200000 as the value: `compact` gives 1.2M, `compactCurrency` gives $1.2M, `compactLong` gives 1.2 million, `currency` gives USD1,200,000.00, `decimalPattern` gives 1,200,000, `percentPattern` gives 120,000,000%, `scientificPattern` gives 1E6, and `simpleCurrency` gives $1,200,000. A `DateTime` placeholder takes a `DateFormat` constructor name such as `yMd` or `yMMMd`, so `7/9/1959` in US English becomes `9.07.1959` in Russian.

## Right-to-Left and Layout

Adding `ar` or `he` flips the app automatically once the locale is supported. Your code keeps that working by using direction-aware values: `EdgeInsetsDirectional.only(start: 16)` rather than `EdgeInsets.only(left: 16)`, `AlignmentDirectional.centerStart`, and `Directionality.of(context)` when you must branch. German and Finnish strings run about 30% longer than English, so check `Row`s with long labels.

## Letting Users Pick a Language

`MaterialApp.locale` overrides the system language. Keep the choice in your state layer and persist it. For iOS, add the supported locales to `CFBundleLocalizations` in `ios/Runner/Info.plist`, otherwise the App Store and the OS language settings won't list them.

## Verify

```bash
flutter gen-l10n
flutter analyze
grep -rnE "Text\('[A-Za-z]" lib --include=*.dart | grep -v "l10n/"     # hard-coded strings left
```

Set `untranslated-messages-file: untranslated.json` in `l10n.yaml`, and fail CI when the file lists anything. Test the messages that have logic:

```dart
// test/summary_screen_test.dart
import 'package:expense_tracker/l10n/app_localizations.dart';
import 'package:expense_tracker/main.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';

void main() {
  testWidgets('plural and currency render in Spanish', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        locale: Locale('es'),
        localizationsDelegates: [
          AppLocalizations.delegate,
          ...GlobalMaterialLocalizations.delegates,
        ],
        supportedLocales: AppLocalizations.supportedLocales,
        home: SummaryScreen(),
      ),
    );
    await tester.pumpAndSettle();

    expect(find.text('3 gastos'), findsOneWidget);
  });
}
```

## Gotchas

1. **`Target of URI doesn't exist: package:flutter_gen/...`.** Symptom: an analyzer error after upgrading Flutter. Cause: the synthetic package is gone. Fix: follow the upgrade steps in Step 0.
2. **`Null check operator` from `AppLocalizations.of(context)`.** Symptom: a crash on the first frame. Cause: it ran above `MaterialApp`. Fix: use `onGenerateTitle`, and read localizations below the app widget.
3. **`No MaterialLocalizations found`.** Symptom: a date picker throws. Cause: the Material delegates are missing. Fix: add `...GlobalMaterialLocalizations.delegates`.
4. **ARB generation fails with a syntax error.** Symptom: `flutter gen-l10n` stops on a placeholder. Cause: a trailing comma, or a placeholder not declared in the `@key` block. Fix: fix the JSON, and declare it.
5. **Select always shows `other`.** Symptom: the wrong label, without an error. Cause: select matching is case sensitive. Fix: pass the exact key.
6. **The app ignores the device language on iOS.** Symptom: English on a Spanish iPhone. Cause: `CFBundleLocalizations` lacks the locale. Fix: add it to `Info.plist`.

## Anti-Patterns

- Concatenating translated pieces. Word order changes across languages, so use one message with placeholders.
- Hard-coding `'$count items'`. Use a plural.
- Formatting currency by hand. Use `format`, which knows symbols and grouping.
- Using `EdgeInsets.only(left:)` in a translated app. Use the directional form.

## Pre-Merge Checklist

- [ ] No user-visible string is hard-coded
- [ ] Every message has an `@key` description for translators
- [ ] Plurals, dates and currencies use ARB formats
- [ ] The untranslated report is empty
- [ ] Checked once with a right-to-left locale and once with a long-string locale

## See Also

- `flutter-theme` for typography per script
- `flutter-build-responsive-layout` for text that grows
- `flutter-add-widget-test` for locale tests
