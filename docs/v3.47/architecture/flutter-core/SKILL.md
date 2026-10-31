---
name: flutter-core
description: Sets up the architecture of a production Flutter app - feature-first Clean Architecture, Riverpod 3 code generation, typed go_router routes with an auth guard that never rebuilds the router, GetIt and injectable, Freezed models, fpdart errors and the material_ui migration. Use first on any Flutter project, when scaffolding an app, adding the first real feature, choosing state management, or moving from Provider or GetX to Riverpod.
metadata:
  tags:
  - flutter
  - architecture
  - riverpod
  - go_router
  - freezed
  - dart
---
# Flutter Core

Every other Flutter decision sits on this layer: where code lives, how state moves, how a screen is reached and how a failure travels. This skill sets those four up once, so a feature added a year from now looks like the first one. Business logic stays out of widgets, navigation is checked at compile time, models are immutable, and errors are values rather than exceptions.

Use it before any other Flutter skill when you scaffold a project, add the first real feature to an app without architecture, or migrate from Provider v4 or GetX to Riverpod. Skip it when a project already follows these conventions, when you're making isolated UI changes, or when you're chasing a runtime bug unrelated to architecture. Never re-architect a working app to match this page.

## Step 0: Read the Versions You Are On

A project's own SDK decides which package majors it can take, so read it before following any pin here:

```bash
flutter --version
dart pub outdated --no-dev-dependencies
grep -nE "^\s+(flutter_riverpod|riverpod_annotation|go_router|freezed|material_ui|build_runner):" pubspec.yaml
```

The pins here were read from pub.dev on 2026-10-30 against Flutter 3.47.9 and Dart 3.13.7, and every example on this page and in `references/` was compiled against them. Riverpod 3 needs `riverpod_generator` 4, and Freezed 4 needs Dart 3.13. Everything here was compiled, not run on a device, so walk the sign-in and routing flows on real hardware.

`build_runner: ^2.16.1` sits below the latest 2.17.1 on purpose: 2.17.1 needs `build` 5, which `envied_generator`, `hive_ce_generator`, `injectable_generator`, `isar_community_generator` and `riverpod_generator` don't accept yet, and 2.16.2 needs `analyzer` 14.3, which `injectable_generator` 3.1.3 can't take. `^2.17.1` ends in `version solving failed`; the wider range lets the solver pick the newest release a project can take.

### Material moved out of the SDK: material_ui and cupertino_ui

Flutter 3.47 published Material and Cupertino as the packages `material_ui` and `cupertino_ui`. The SDK's own copies no longer change, and Flutter plans to deprecate them formally in a coming stable release. `go_router` 18, `pinput` 7, `shimmer` 4, `animations` 3, `google_fonts` 9 and `dynamic_color` 2 already import `material_ui`.

Migrate before you take those majors:

```bash
dart fix --apply --code=migrate_design_widgets
# when the pubspec step fails, add it yourself and run the fix again
flutter pub add material_ui
dart fix --apply
```

The fix rewrites `package:flutter/material.dart` to `package:material_ui/material_ui.dart`, and the same applies to Cupertino. Set `localizationsDelegates: GlobalMaterialLocalizations.delegates` to cover Material, Cupertino and Widgets strings. When a dependency still imports the SDK copy, wrap it in `MaterialUiCompatibilityBridge`, a deprecated migration aid; the bridge can't fix a dependency whose public API exposes SDK types, so that dependency has to migrate first. A project that stays on the SDK copy stays on the previous major of those packages. The examples also import `firebase_core: ^5.1.0`.

## Choosing the Foundation

| Decision | Pick | When another option wins |
|---|---|---|
| App state | Riverpod 3 with code generation: compile-safe providers and no `BuildContext` needed | Bloc when a team already runs it and wants explicit event logs. Provider only for a small app that's staying small |
| Navigation | `go_router` + `go_router_builder`: typed routes, one redirect guard, deep links | Plain `Navigator` for a two-screen tool with no links |
| Dependency injection | GetIt + `injectable` for data-layer singletons that exist outside the widget tree | Riverpod alone when nothing runs outside the tree, such as a background isolate or a push handler |
| Models | Freezed: immutable classes with `copyWith`, unions and JSON | Dart records for two-field private returns; `built_value` only in a codebase already on it |
| Errors | fpdart `Either<Failure, T>` from repositories: the compiler makes callers handle failure | Exceptions inside the data layer, where they're caught and mapped |

## The Architecture Stack

| Layer | Responsibility | No-import rule |
|---|---|---|
| `presentation/` | Widgets, UI state, providers | Never imports `data/` directly |
| `domain/` | Entities, repo interfaces, use cases | Never imports Flutter or `data/` |
| `data/` | Repo implementations, DTOs, remote/local sources | Never imports `presentation/` |

Domain is pure Dart with zero Flutter dependencies. Breaking one of these rules is an architecture failure, not a style issue. Each feature is a self-contained unit:

```
lib/
  features/
    auth/
      data/
        datasources/        # remote_datasource.dart, local_datasource.dart
        models/             # user_model.dart (DTO with fromJson/toJson)
        repositories/       # auth_repository_impl.dart
      domain/
        entities/           # user.dart (pure Dart entity)
        repositories/       # auth_repository.dart (abstract interface)
        usecases/           # sign_in_usecase.dart, sign_out_usecase.dart
      presentation/
        pages/              # login_page.dart
        widgets/            # login_form.dart, biometric_button.dart
        providers/          # auth_provider.dart (Riverpod)
  core/
    analytics/              # AnalyticsService, GoRouterObserver
    errors/                 # failure.dart, exceptions.dart, grpc_error_mapper.dart
    network/                # DioClient, GrpcClient, ApiClient, interceptors/
    payments/               # PaymentService (facade over RevenueCat / Stripe)
    performance/            # FrameBudgetMonitor, MemoryPressureNotifier
    router/                 # router.dart (go_router)
    di/                     # injection.dart (GetIt config)
    theme/                  # app_theme.dart
    constants/              # app_constants.dart
  shared/
    widgets/                # reusable widgets across features
    utils/                  # date_utils.dart, string_utils.dart
```

Files are `snake_case.dart`, classes are `PascalCase`, providers are `camelCase` (the generator does this for you), and feature folders are singular nouns: `auth/`, not `authentication/`.

## State Management (Riverpod Code-Gen)

Riverpod with `riverpod_annotation` code generation is the state layer, and every provider is annotated.

```yaml
dependencies:
  flutter_riverpod: ^3.4.3
  riverpod_annotation: ^4.0.7
  hooks_riverpod: ^3.4.3
  flutter_hooks: ^0.21.3+1
  go_router: ^18.1.2
  get_it: ^9.3.0
  injectable: ^3.0.0
  freezed_annotation: ^3.1.0
  json_annotation: ^4.12.0
  fpdart: ^1.2.0
  material_ui: ^1.9.0

dev_dependencies:
  riverpod_generator: ^4.0.9
  go_router_builder: ^4.5.3
  injectable_generator: ^3.1.3
  freezed: ^4.0.3
  json_serializable: ^6.15.2
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
```

Use `ref.watch` inside `build()` only, `ref.read` inside callbacks and event handlers, and `ref.listen` for side effects such as navigation and snackbars. Providers auto-dispose by default; use `@Riverpod(keepAlive: true)` only for singletons. Run `dart run build_runner build` after every annotation change. Provider patterns, widget consumption and hooks are in `references/riverpod-patterns.md`.

### Riverpod 3 breaking changes to watch for

`Ref` is one type now: every generated subclass such as `UserListRef` is gone, so functional providers take `Ref ref`. `FamilyNotifier` and its variants are gone too; a plain `Notifier` takes family arguments through its generated `build` signature, and those arguments are no longer readable from `Ref`. `AsyncValue.valueOrNull` was removed, and `.value` now returns `null` during errors. `StateProvider`, `StateNotifierProvider` and `ChangeNotifierProvider` moved to `package:flutter_riverpod/legacy.dart`. Providers compare values with `==` to filter updates, a `Ref` or notifier throws when used after disposal (only `mounted` stays safe), and a paused provider's `StreamSubscription` is paused with it. `ProviderObserver` methods now take a `ProviderObserverContext`. `family.overrideWith` is deprecated in favour of `overrideWith2`.

## Worked Example: Sign-In Through Every Layer

One feature crosses all three layers below. Copy the shape for every feature after it.

### Failures and the error boundary with fpdart

Repository methods return an fpdart `Either<Failure, T>` and never throw. Exceptions stop at the repository, where they become one sealed type every caller has to handle.

```dart
// lib/core/errors/failure.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'failure.freezed.dart';

@freezed
sealed class Failure with _$Failure {
  const factory Failure.network(String message) = NetworkFailure;
  const factory Failure.server(int statusCode, String message) = ServerFailure;
  const factory Failure.cache(String message) = CacheFailure;
  const factory Failure.auth(String message) = AuthFailure;
}
```

### Immutable models with Freezed

A domain entity knows nothing about JSON, while its DTO in `data/` carries the wire names and converts itself with `toEntity()`. A Freezed class must be `sealed` or `abstract`.

```dart
// lib/features/auth/domain/entities/user.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';

@freezed
abstract class User with _$User {
  const factory User({
    required String id,
    required String email,
    required String displayName,
  }) = _User;
}
```

```dart
// lib/features/auth/data/models/user_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';

import '../../domain/entities/user.dart';

part 'user_model.freezed.dart';
part 'user_model.g.dart';

@freezed
abstract class UserModel with _$UserModel {
  const UserModel._();

  const factory UserModel({
    required String id,
    required String email,
    @JsonKey(name: 'display_name') required String displayName,
  }) = _UserModel;

  factory UserModel.fromJson(Map<String, dynamic> json) =>
      _$UserModelFromJson(json);

  User toEntity() => User(id: id, email: email, displayName: displayName);
}
```

Freezed 4 no longer accepts the `final` keyword inside a factory constructor's parameter list, because Dart 3.13 rejects that syntax. Freezed 3.0 removed `when` and `map`, and 3.1 added them back, so they exist in every current release. Prefer a switch expression anyway: it's exhaustive at compile time, and a new union case breaks the build instead of a screen.

### Domain: the repository contract and use case

```dart
// lib/features/auth/domain/repositories/auth_repository.dart
import 'package:fpdart/fpdart.dart';

import '../../../../core/errors/failure.dart';
import '../entities/user.dart';

abstract interface class AuthRepository {
  Future<Either<Failure, User>> signIn(String email, String password);
}
```

```dart
// lib/features/auth/domain/usecases/sign_in_usecase.dart
import 'package:fpdart/fpdart.dart';
import 'package:injectable/injectable.dart';

import '../../../../core/errors/failure.dart';
import '../entities/user.dart';
import '../repositories/auth_repository.dart';

class SignInParams {
  const SignInParams({required this.email, required this.password});
  final String email;
  final String password;
}

@injectable
class SignInUseCase {
  const SignInUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<Failure, User>> call(SignInParams params) =>
      _repository.signIn(params.email, params.password);
}
```

### Data: mapping exceptions to Failure

`AuthRepositoryImpl` is the one place an exception turns into a `Failure`: it catches `DioException`, maps 401 to `Failure.auth`, other statuses to `Failure.server` and no response to `Failure.network`. The full class is in `references/data-layer.md`.

### Dependency injection with GetIt and injectable

GetIt holds the data-layer singletons, and injectable generates their registration. `@lazySingleton` makes one instance on first use, `@singleton` one at app start, `@injectable` a new instance per injection, and `@LazySingleton(as: Interface)` binds an implementation to its interface.

```dart
// lib/core/di/injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit(initializerName: 'init')
Future<void> configureDependencies() async => getIt.init();
```

### Presentation: the notifier and its bridge to GetIt

`class AuthNotifier` generates `authProvider`, because `riverpod_generator` strips the Notifier suffix. A one-line provider is the only place Riverpod reaches into GetIt. Reading a union with a switch is shown in `authLabel`.

```dart
// lib/features/auth/presentation/providers/auth_provider.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../../../core/di/injection.dart';
import '../../../../core/errors/failure.dart';
import '../../domain/entities/user.dart';
import '../../domain/usecases/sign_in_usecase.dart';

part 'auth_provider.freezed.dart';
part 'auth_provider.g.dart';

@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.unauthenticated() = AuthStateUnauthenticated;
  const factory AuthState.loading() = AuthStateLoading;
  const factory AuthState.authenticated(User user) = AuthStateAuthenticated;
  const factory AuthState.error(String message) = AuthStateError;
}

String authLabel(AuthState state) => switch (state) {
  AuthStateUnauthenticated() => 'Signed out',
  AuthStateLoading() => 'Signing in',
  AuthStateAuthenticated(:final user) => user.displayName,
  AuthStateError(:final message) => message,
};

@riverpod
SignInUseCase signInUseCase(Ref ref) => getIt<SignInUseCase>();

@Riverpod(keepAlive: true)
class AuthNotifier extends _$AuthNotifier {
  @override
  AuthState build() => const AuthState.unauthenticated();

  Future<void> signIn(String email, String password) async {
    state = const AuthState.loading();
    final result = await ref.read(signInUseCaseProvider)(
      SignInParams(email: email, password: password),
    );
    state = result.fold(
      (failure) => AuthState.error(switch (failure) {
        NetworkFailure(:final message) => message,
        ServerFailure(:final message) => message,
        CacheFailure(:final message) => message,
        AuthFailure(:final message) => message,
      }),
      AuthState.authenticated,
    );
  }

  void signOut() => state = const AuthState.unauthenticated();
}
```

The login screen, typed routes and the guard are in `references/routing-and-login-screen.md`. Read that file before writing a router: the guard has one trap that resets the whole navigation stack.

## Navigation With go_router and Type Safety

`go_router` with `go_router_builder` gives compile-time route safety, so a broken route fails the build. Calls look like `const HomeRoute().go(context)`, `const ProfileRoute(userId: '123').push(context)` and `context.pop()`. Never use `Navigator.push` directly: it skips the auth guard and the transition system.

`go_router_builder` 4 requires the public `$RouteName` mixin on every route class, and the mixin supplies `go`, `push` and `location`. Since `go_router` 15, URLs are case sensitive (`caseSensitive: false` restores the old matching). Since 17, `ShellRoute` navigation notifies the `GoRouter` observers; pass `notifyRootObserver: false` to opt out. Version 16.3 added a top-level `onEnter` callback for decisions that need both the current and the next route.

## Verify

Run these from the project root; each must finish clean.

```bash
dart run build_runner build
flutter analyze
flutter test
grep -rnE "package:flutter/(material|cupertino)\.dart" lib test     # expect nothing after the migration
grep -rln "package:flutter" lib/features/*/domain                    # expect nothing: domain is pure Dart
grep -rln "/data/" lib/features/*/presentation                       # expect nothing: no layer leak
grep -rn "Navigator.push" lib                                        # expect nothing
```

Then prove the guard with the router test in `references/testing-setup.md`: a signed-out user lands on login, signing in reaches home, and the `GoRouter` object stays identical. The same file has the use case tests, a retry test and the layout of `test/`.

## Gotchas

1. **`version solving failed` naming `build_runner` and `build`.** Symptom: `pub get` fails after you copy the latest versions. Cause: `build_runner` 2.17 needs `build` 5, and generators such as `injectable_generator` 3.1.3 still need `build` 4. Fix: pin `build_runner: ^2.16.1` until they catch up.
2. **The app jumps to `/` and the back stack is gone after sign-in.** Symptom: every auth change resets navigation, and loading states flicker. Cause: `ref.watch(authProvider)` inside the router provider builds a new `GoRouter`. Fix: keep one router, feed `refreshListenable`, and `ref.read` in `redirect`.
3. **`'string' OR 'number' must be set as the value of the parameter`.** Symptom: an assert in debug from `logEvent`. Cause: Firebase Analytics accepts only `String` and `num`, so a `bool` fails. Fix: send `success ? 1 : 0`.
4. **`Undefined class 'UserListRef'`.** Symptom: every functional provider stops compiling after the upgrade. Cause: Riverpod 3 has one `Ref` type. Fix: declare `Ref ref`.
5. **`No Material widget found` on Flutter 3.47.** Symptom: a package widget throws inside a `Scaffold` that's clearly there. Cause: the widget checks the SDK's Material and your app uses `material_ui`, or the reverse. Fix: migrate imports, update the package, and use `MaterialUiCompatibilityBridge` only as a stopgap.
6. **A customer is charged twice after a timeout.** Symptom: duplicate orders in the logs. Cause: a retry interceptor re-sent a `POST`. Fix: retry idempotent methods only, as in `references/api-client.md`.
7. **`Using "ref" when a widget is about to or has been unmounted is unsafe`.** Symptom: a red screen after `await` or `Navigator.pop`. Cause: `ref` belongs to a disposed widget. Fix: read the notifier before the gap, or check `context.mounted`.
8. **A link that used to work now shows the error page.** Symptom: `/Profile/1` fails after moving past `go_router` 15. Cause: URLs became case sensitive. Fix: correct the link, or set `caseSensitive: false`.
9. **`The method 'when' isn't defined` on a Freezed class.** Symptom: unions lost their helpers. Cause: the project resolved Freezed 3.0, which removed them. Fix: take a current Freezed, or use a switch expression.

## Anti-Patterns

| Don't | Do instead |
|---|---|
| Call a repository from a widget | Widget → provider → use case → repository |
| Return `null` or throw from a repository | Return `Either<Failure, T>` |
| Import `dio` or Flutter in `domain/` | Keep domain pure and map in `data/` |
| Share one model across wire, storage and UI | A DTO in `data/` and an entity in `domain/`, joined by `toEntity()` |
| `ref.watch` inside a callback | `ref.read` there; `ref.watch` only in `build()` |
| Build a `GoRouter` in a provider that watches auth | One keep-alive router with `refreshListenable` |
| Pass raw path strings to `context.go` | Typed routes |
| `keepAlive: true` on everything | Auto-dispose by default, keep-alive for singletons |

## Pre-Merge Checklist

- [ ] `flutter analyze` and `flutter test` pass, and generated code is current
- [ ] No `package:flutter/material.dart` or `cupertino.dart` import is left
- [ ] `domain/` imports no Flutter and no `data/`
- [ ] Every repository method returns `Either<Failure, T>`
- [ ] Every union read uses an exhaustive switch
- [ ] Every new screen has a typed route that passes through the guard
- [ ] The router provider is `keepAlive` and watches nothing
- [ ] Retries touch idempotent methods only
- [ ] Analytics parameters are `String` or `num`, and hold no personal data

## See Also

`flutter-state` (app-wide state, theme, connectivity), `flutter-network` (transports, token refresh), `flutter-database` (local storage, sync), `flutter-setup-declarative-routing` (deeper go_router), `flutter-analytics-firebase` (analytics events), `flutter-performance` (frame budget), `flutter-security` (hardening before release).

Depth in this skill: `references/riverpod-patterns.md`, `references/routing-and-login-screen.md`, `references/data-layer.md`, `references/api-client.md`, `references/analytics-and-crash-reporting.md`, `references/testing-setup.md`.
