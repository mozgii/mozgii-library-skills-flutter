---
name: flutter-apply-architecture-best-practices
description: Structures a Flutter app the way the Flutter team recommends - a UI layer of views and view models, a data layer of repositories and services, an optional domain layer, one-way data flow, commands for user actions, fakes for tests. Use when starting a new project, refactoring a screen that mixes UI and data, deciding where code belongs, adding a repository, or asking what MVVM looks like in Flutter.
metadata:
  tags:
  - flutter
  - architecture
  - mvvm
---
# Flutter App Architecture

Most Flutter apps fail slowly: a widget fetches its own data, a second screen copies that code, and then nothing can be tested. Dividing the app in two, a UI layer above a data layer, with data flowing one way, stops that. This skill shows the structure Flutter's own architecture guide recommends, with `ChangeNotifier` view models. `flutter-core` shows the same layers with Riverpod and Clean Architecture.

## Step 0: See What the Project Already Does

```bash
flutter --version
grep -nE "^\s+(provider|flutter_riverpod|flutter_bloc|get_it|freezed|built_value):" pubspec.yaml
ls lib lib/*
```

Follow the pattern a project already uses. Migrate to another one only when you're asked to. The examples also import `http: ^1.6.0`.

## The Recommendations and Their Weight

| Practice | Weight | In one line |
|---|---|---|
| Separate UI and data layers | Strongly recommended | Each class has one responsibility |
| Repository pattern | Strongly recommended | Repositories and services isolate data access |
| MVVM in the UI layer | Strongly recommended | Views stay dumb, view models hold logic |
| No logic in widgets | Strongly recommended | Only simple conditionals, animation, layout and simple routing |
| One-way data flow | Strongly recommended | State flows from data to UI, events the other way |
| Immutable models | Strongly recommended | A change creates a new instance |
| Dependency injection | Strongly recommended | No globally reachable objects |
| Abstract repositories | Strongly recommended | Swap implementations per environment |
| Fakes in tests | Strongly recommended | Test inputs and outputs, not internals |
| Commands for user events | Recommended | A standard way to send events, safe to render |
| `freezed` or `built_value` | Recommended | Generated models, at some build-time cost |
| `go_router` | Recommended | The default for most apps |
| `ChangeNotifier` and listenables | Conditional | State management is partly preference |
| Domain layer and use cases | Conditional | Only when logic crowds view models or repeats |
| Separate API and domain models | Conditional | Pays off in large apps, more verbose |

## Which State Tool for the View Model

| Your situation | Choose | Why the others lose here |
|---|---|---|
| Small team, few dependencies wanted | `ChangeNotifier` + `provider` | Riverpod's code generation is more than you need |
| Async data everywhere, a large app | Riverpod (`flutter-core`) | A hand-rolled `ChangeNotifier` repeats loading and error handling |
| A team that already runs BLoC | `flutter_bloc` | Mixing a second tool costs more than either tool saves |

## The Layers

The **view** is a widget. It reads state from one view model and forwards user events to it. A view may hold a simple `if` for showing or hiding, animation code, layout that depends on screen size, and basic routing.

The **view model** is a plain class exposing immutable state and methods the view calls. It gets its repositories through the constructor and knows nothing about widgets, which is why it can be tested without Flutter.

Each kind of data has one source of truth, its **repository**. It consumes services, turns raw API models into domain models, and owns caching, retries and offline behaviour.

The **service** is a thin, stateless wrapper over one outside system: an HTTP API, a local database, a platform plugin. It returns raw models or a result type.

The optional **use case** sits between view models and repositories. Add one only when logic merges several repositories, or when two view models need the same logic. In most apps it adds overhead.

## Project Layout

Group the UI by feature and the data by type, so a repository can serve several features.

```text
lib/
├── data/
│   ├── repositories/    # expense_repository.dart, expense_repository_remote.dart
│   ├── services/        # expense_api.dart, receipt_store.dart
│   └── models/          # expense_dto.dart (API shapes)
├── domain/
│   ├── models/          # expense.dart (what the app works with)
│   └── use_cases/       # only when justified
├── ui/
│   ├── core/            # shared widgets and theme
│   └── expenses/
│       ├── expense_list_view_model.dart
│       └── expense_list_screen.dart
└── main.dart
```

Name classes after their role: `ExpenseListViewModel`, `ExpenseRepository`. Use `ui/core/` for shared widgets, because a folder called `widgets` reads like Flutter's own library.

## Worked Example: Listing Expenses

The sealed `Result` means a failure can't be forgotten: the view model has to handle it to compile.

```dart
// lib/domain/result.dart
sealed class Result<T> {
  const Result();
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Failed<T> extends Result<T> {
  const Failed(this.error);
  final Exception error;
}
```

```dart
// lib/domain/models/expense.dart
class Expense {
  const Expense({required this.id, required this.note, required this.cents});

  final String id;
  final String note;
  final int cents;
}
```

```dart
// lib/data/services/expense_api.dart
import 'dart:convert';
import 'dart:io';

import 'package:http/http.dart' as http;

class ExpenseApi {
  ExpenseApi(this._client, this._base);

  final http.Client _client;
  final Uri _base;

  Future<List<Map<String, dynamic>>> fetchAll() async {
    final response = await _client.get(_base.resolve('expenses'));
    if (response.statusCode != HttpStatus.ok) {
      throw HttpException('expenses: ${response.statusCode}');
    }
    return (jsonDecode(response.body) as List).cast<Map<String, dynamic>>();
  }
}
```

```dart
// lib/data/repositories/expense_repository.dart
import '../../domain/models/expense.dart';
import '../../domain/result.dart';
import '../services/expense_api.dart';

abstract interface class ExpenseRepository {
  Future<Result<List<Expense>>> expenses({bool refresh = false});
}

class ExpenseRepositoryRemote implements ExpenseRepository {
  ExpenseRepositoryRemote(this._api);

  final ExpenseApi _api;
  List<Expense>? _cached;

  @override
  Future<Result<List<Expense>>> expenses({bool refresh = false}) async {
    final cached = _cached;
    if (cached != null && !refresh) return Ok(cached);
    try {
      final rows = await _api.fetchAll();
      final fresh = [
        for (final row in rows)
          Expense(
            id: row['id'] as String,
            note: row['note'] as String,
            cents: row['amount_cents'] as int,
          ),
      ];
      return Ok(_cached = fresh);
    } on Exception catch (e) {
      return Failed(e);
    }
  }
}
```

```dart
// lib/ui/expenses/expense_list_view_model.dart
import 'package:flutter/foundation.dart';

import '../../data/repositories/expense_repository.dart';
import '../../domain/models/expense.dart';
import '../../domain/result.dart';

class ExpenseListViewModel extends ChangeNotifier {
  ExpenseListViewModel(this._repository);

  final ExpenseRepository _repository;

  List<Expense> _expenses = const [];
  List<Expense> get expenses => _expenses;

  bool _loading = false;
  bool get loading => _loading;

  String? _error;
  String? get error => _error;

  int get totalCents => _expenses.fold(0, (sum, e) => sum + e.cents);

  Future<void> load({bool refresh = false}) async {
    _loading = true;
    _error = null;
    notifyListeners();

    switch (await _repository.expenses(refresh: refresh)) {
      case Ok(:final value):
        _expenses = value;
      case Failed():
        _error = 'Could not load your expenses';
    }

    _loading = false;
    notifyListeners();
  }
}
```

A project from `flutter create` still imports `package:flutter/material.dart`. Move it first: run `flutter pub add material_ui`, then `dart fix --apply --code=migrate_design_widgets`, which rewrites the imports in `lib/` and `test/`. Flutter's [migration guide](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui) covers the rest.

```dart
// lib/ui/expenses/expense_list_screen.dart
import 'package:material_ui/material_ui.dart';

import 'expense_list_view_model.dart';

class ExpenseListScreen extends StatelessWidget {
  const ExpenseListScreen({super.key, required this.viewModel});

  final ExpenseListViewModel viewModel;

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Expenses')),
        body: ListenableBuilder(
          listenable: viewModel,
          builder: (context, _) {
            if (viewModel.loading && viewModel.expenses.isEmpty) {
              return const Center(child: CircularProgressIndicator());
            }
            if (viewModel.error case final message?) {
              return Center(
                child: TextButton(
                  onPressed: () => viewModel.load(refresh: true),
                  child: Text('$message. Try again'),
                ),
              );
            }
            if (viewModel.expenses.isEmpty) {
              return const Center(child: Text('No expenses yet'));
            }
            return RefreshIndicator(
              onRefresh: () => viewModel.load(refresh: true),
              child: ListView.builder(
                itemCount: viewModel.expenses.length,
                itemBuilder: (context, index) {
                  final expense = viewModel.expenses[index];
                  return ListTile(
                    title: Text(expense.note),
                    trailing: Text((expense.cents / 100).toStringAsFixed(2)),
                  );
                },
              ),
            );
          },
        ),
      );
}
```

Create the view model once, above the screen, and call `load()` there, for instance in a `Provider` factory or a route's builder. Calling it inside `build` would refetch on every rebuild. A **command** wraps one action such as `load` in an object holding its running, error and completed state, so a view binds to it instead of juggling flags. Add one when a screen has several actions that can run at once.

## Adding a Feature, In Order

Work from the data outward: domain model, service, repository (interface first), then the view model, its view, and finally the wiring in your injection setup, such as `provider` or `get_it`. Add a use case only if the rule above applies. This order keeps every layer testable before the next exists.

## Verify

Test the view model against a fake repository. A fake is plain code, so it needs no mocking package.

```dart
// test/ui/expenses/expense_list_view_model_test.dart
import 'package:app/data/repositories/expense_repository.dart';
import 'package:app/domain/models/expense.dart';
import 'package:app/domain/result.dart';
import 'package:app/ui/expenses/expense_list_view_model.dart';
import 'package:flutter_test/flutter_test.dart';

class FakeExpenseRepository implements ExpenseRepository {
  Result<List<Expense>> next = const Ok([]);

  @override
  Future<Result<List<Expense>>> expenses({bool refresh = false}) async => next;
}

void main() {
  test('a failure sets an error and keeps the list empty', () async {
    final repository = FakeExpenseRepository()..next = Failed(Exception('down'));
    final viewModel = ExpenseListViewModel(repository);

    await viewModel.load();

    expect(viewModel.error, isNotNull);
    expect(viewModel.expenses, isEmpty);
    expect(viewModel.loading, isFalse);
  });
}
```

```bash
flutter test
grep -rn "package:http\|jsonDecode" lib/ui            # expect nothing: the UI never reaches the network
```

## Gotchas

1. **The screen refetches on every keystroke.** Symptom: a network call per rebuild. Cause: `load()` runs inside `build`. Fix: call it once where the view model is created.
2. **The UI doesn't update.** Symptom: state changes, the screen stays the same. Cause: a list mutated in place, or a missing `notifyListeners()`. Fix: assign a new list, then notify.
3. **`A ChangeNotifier was used after being disposed`.** Symptom: an error when leaving a screen mid-request. Cause: an async call finished after `dispose`. Fix: track disposal and skip the notify, or cancel the work.
4. **One repository per screen.** Symptom: two screens show different data. Cause: repositories were split by screen. Fix: one repository per data type, shared across features.
5. **Build times creep up.** Symptom: `build_runner` takes minutes. Cause: `freezed` on every model. Fix: use it where equality and `copyWith` earn their cost.

## Anti-Patterns

- A widget calling `http.get`. Move it into a service behind a repository.
- A repository returning an API model. Convert it to a domain model first.
- A global singleton for app state. Inject it through a constructor.
- A use case that only forwards a call. Delete it.

## Pre-Merge Checklist

- [ ] No widget imports a service or a repository implementation
- [ ] View models take their dependencies through the constructor
- [ ] Models are immutable
- [ ] Each screen shows loading, error, empty and filled states
- [ ] Repositories and view models have unit tests with fakes

## See Also

- `flutter-core` for the Riverpod and Clean Architecture stack
- `flutter-use-http-package` for the service layer
- `flutter-database` for local repositories
- `flutter-add-widget-test` for testing views
