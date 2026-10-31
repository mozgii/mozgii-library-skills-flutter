---
title: "Flutter Architecture Conventions"
author: MOZGII, Inc.
lang: en
kind: rule
description: "Feature-first folder structure, Clean Architecture layer separation (data/domain/presentation), widget ownership and the shared-widget-first order, naming conventions, import discipline, file size limits, prohibited patterns, gRPC layer placement, adaptive layout rules, and payment layer placement for all Flutter projects."
tags: ["flutter", "architecture", "clean-architecture", "conventions", "grpc", "analytics", "payments", "foldable", "adaptive", "shared-widgets", "reuse"]
---
# Flutter Architecture Conventions

Applies to all Dart source files in `lib/`. Governs folder structure, layer separation, naming conventions, and import discipline for all Flutter projects. These rules enforce Clean Architecture: data / domain / presentation: with zero cross-layer leakage.

---

## Folder Structure

Use **feature-first** organization. Every feature is self-contained.

```
lib/
  features/
    <feature>/
      data/
        datasources/        # remote and local data sources
        models/             # DTOs with fromJson/toJson
        repositories/       # repository implementations
      domain/
        entities/           # pure Dart entities (no Flutter, no json)
        repositories/       # abstract repository interfaces
        usecases/           # one class per use case
      presentation/
        pages/              # full screens (one per route)
        widgets/            # feature-specific widgets
        providers/          # Riverpod providers / Bloc cubits
  core/
    di/                     # GetIt injection config
    errors/                 # Failure sealed class, exceptions
    network/                # Dio client, interceptors
    router/                 # go_router config and routes
    theme/                  # ThemeData, ThemeExtension
    constants/              # app-wide constants, spacing
  shared/
    ui/                     # reusable widgets across features, the component layer
      buttons/  inputs/  layout/  feedback/  media/  actions/  marks/
    branding/               # logo, wordmark, splash, loader
    fx/                     # shared visual effects
    domain/                 # cross-feature domain types
    utils/                  # pure utility functions
```

**The component layer is `shared/ui/`, not `shared/widgets/`.** Group by role
(`buttons/`, `inputs/`, `layout/`, `feedback/`) rather than dropping every widget in one
flat folder: a flat folder stops being searchable at about thirty files, and unsearchable
is the same as absent to whoever is about to write the duplicate.

**Rule**: A feature folder must contain all three layers (`data/`, `domain/`, `presentation/`). If a feature uses only two layers, it is not a feature: move it to `shared/` or `core/`.

---

## Widget Ownership (Non-Negotiable)

**Reuse is the default path. Authoring a new widget is the last branch, not the first.**

```
1. Enumerate  what already exists in shared/ui/
2. Reuse      it fits as-is                        → use it, write nothing
3. Extend     it nearly fits                       → add an optional parameter
4. Fork       extension cannot be made safe        → a named constructor or a sibling
5. Build      nothing comparable exists            → author it, in the right folder
```

**A widget is never authored before step 1 has been run.** "I did not see one" is not step 1.

### Where a widget lives: the count decides

| Used in | Home |
|---|---|
| exactly 1 place | `features/<feature>/presentation/widgets/` |
| **2 or more places** | **`shared/ui/`** |

**Two is the threshold, and it is not a target to grow into.** A control you are about to
use a second time is built shared *now*, not promoted later: "later" is the name of the
second copy, and by the time anyone notices, the two have drifted.

This is deliberately stricter than the Rule of Three. A mobile binary is frozen by store
review: duplication found after release is fixed on a release cycle, not a deploy.

### How to enumerate call sites

```bash
grep -rn "PrimaryButton" lib/
```

A widget is invoked by its class name, a unique PascalCase symbol, so a literal text search is the right tool: it finds every call site, including ones in tests. Check that `grep` itself found the widget's declaration, since a typo in the name also returns nothing, and zero results read as "this widget is unused".

### Extending an existing shared widget

Every existing call site must keep compiling **and keep rendering identically**.

| Change | Verdict |
|---|---|
| new **optional** named parameter, default reproduces today's render | safe: this is the extension path |
| new **`required`** named parameter | breaks loudly at every call site; `dart analyze` catches all of them |
| **changing an existing default value** | **banned**: compiles, passes analyze, silently re-renders every screen |
| renaming or removing a parameter | use the fork path |

Changing a default is the only edit in this procedure that hides. Nothing in Dart's type
system can see it.

### When to fork instead

Fork when the extension would do any of these:

- change an existing default value
- add a **fourth** boolean flag to one constructor
- require the widget to know which screen is calling it

**Shape of a fork:** a named constructor (`PrimaryButton.icon`, Flutter's own convention
for `ElevatedButton.icon`) or a sibling widget built for reuse from its first line. Never
`WidgetV2`, `WidgetNew` or `WidgetCopy`.

### What a shared widget may not do

- **Import from `features/`.** Promoting a widget means removing those imports, not moving
  the file. A `shared/` widget reaching into one feature compiles and violates the layer
  table below.
- **Take a service, repository or provider.** Primitives take props. A widget that resolves
  its own dependencies cannot be previewed or golden-tested in isolation, which makes it
  shared by location only.

### Proving the change was safe

| Step | Skill |
|---|---|
| golden test before the change, and again after | `flutter-add-widget-test` |
| preview both themes | `flutter-add-widget-preview` |
| authoring the widget itself | `flutter-design-ui` |

```bash
dart analyze --fatal-infos --fatal-warnings   # catches every broken call site
flutter test                                  # goldens catch what analyze cannot see
```

A golden that changed is a screen that changed. Never run `--update-goldens` to make a
failure go away: that is the safety net being cut.

---

## Layer Separation (Non-Negotiable)

A widget never calls a repository directly: code in `presentation/` reaches data only through a provider or a
use case in `domain/`, and it never imports `data/`.

| Layer | May import | Must NOT import |
|---|---|---|
| `domain/` | `core/errors/`, `fpdart` | Flutter framework, `data/`, `presentation/` |
| `data/` | `domain/`, `core/`, any package | `presentation/` |
| `presentation/` | `domain/`, `core/`, Flutter | `data/` directly |

**Domain is pure Dart.** If a class in `domain/` imports `package:material_ui/material_ui.dart` or any `package:flutter/` library, it is a violation. Domain logic must be testable without the Flutter framework.

**Presentation never touches data directly.** All data access goes through a repository interface defined in `domain/`.

---

## Naming Conventions

### Files (snake_case)

```
user_repository.dart          # abstract interface
user_repository_impl.dart     # implementation
user_model.dart               # DTO
user.dart                     # domain entity
get_user_usecase.dart         # use case
user_page.dart                # full screen
user_card.dart                # widget
auth_notifier.dart            # Riverpod notifier
```

### Classes (PascalCase)

```dart
class UserRepository {}         // interface
class UserRepositoryImpl {}     // implementation
class UserModel {}              // DTO
class User {}                   // entity
class GetUserUseCase {}         // use case
class UserPage {}               // screen
class UserCard {}               // widget
class AuthNotifier {}           // Riverpod notifier
```

### Providers (camelCase: Riverpod codegen)

```text
@riverpod
Future<User> currentUser(Ref ref) { ... }
// Generated: currentUserProvider

@riverpod
class AuthNotifier extends _$AuthNotifier { ... }
// Generated: authProvider
```

### Files must match their primary class name exactly:
- `UserPage` → `user_page.dart`
- `AuthNotifier` → `auth_notifier.dart`
- `GetUserUseCase` → `get_user_usecase.dart`

---

## File Size Limits

| File type | Maximum lines | Action if exceeded |
|---|---|---|
| Widget (StatelessWidget / ConsumerWidget) | 200 lines | Extract sub-widgets |
| Page (full screen) | 300 lines | Extract widgets and providers |
| Repository implementation | 150 lines | Split into multiple data sources |
| Use case | 50 lines | Consider domain service |
| Provider / Notifier | 100 lines | Split into multiple providers |

---

## Import Discipline

The analyzer rule `directives_ordering` checks import order, and `dart fix --apply --code=directives_ordering` sorts them. `dart format` does not reorder imports. The order is `dart:` first, then `package:` sorted alphabetically, then relative imports. A blank line between sections is the usual style. Flutter and Material have no section of their own: they sort with the other packages.

```text
import 'dart:async';
import 'dart:io';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';
import 'package:myapp/core/errors/failure.dart';
import 'package:myapp/features/auth/domain/entities/user.dart';
```

Enable the check in `analysis_options.yaml`:

```yaml
linter:
  rules:
    directives_ordering: true
```

**Never use** relative `../../` imports crossing feature boundaries. Use `package:myapp/` prefix for cross-feature imports.

---

## Prohibited Patterns

| Pattern | Reason |
|---|---|
| Business logic in `build()` method | Causes unnecessary rebuilds and untestable code |
| `Navigator.push` / `Navigator.pushNamed` | Use `go_router` typed routes only |
| `StatefulWidget` where `ConsumerWidget` suffices | Unnecessary complexity |
| Hardcoded `Colors.*` or `TextStyle(fontSize: 16)` | Must come from `Theme.of(context)` |
| `print()` in production code | Use `debugPrint()` guarded by `kDebugMode` |
| `late` fields without initialization guarantee | Use nullable + null check or initialize in constructor |
| `as` cast without type check | Use `is` check or pattern matching |
| Public API without sealed error type | All public async methods return `Either<Failure, T>` or throw typed exceptions |
| The same control written in two features | It was shared from the moment it was needed twice: `shared/ui/` |
| A `shared/ui/` widget importing `features/` | Shared by location only; re-couples what promoting it decoupled |
| A fourth boolean flag on one widget constructor | 16 render paths, most never tested: fork a named constructor |
| Authoring a widget without enumerating `shared/ui/` first | Produces the duplicate this rule exists to prevent |

---

## Extended Folder Structure

Add to `core/` the following mandatory subdirectories for any app using analytics, networking, or payments:

```
lib/
  core/
    analytics/         # AnalyticsService, GoRouterObserver, event models
    network/           # DioClient, GrpcClient, interceptors/
      interceptors/    # auth_interceptor.dart, retry_interceptor.dart, performance_interceptor.dart
    payments/          # PaymentService facade (wraps Stripe + IAP)
    adaptive/          # FoldAwareLayout, NavigationShell, AdaptiveScaffold
```

**Rule:** `core/analytics/` never imports `features/`. It receives event data as plain typed objects: no feature-specific models.

**Rule:** `core/network/` contains only transport infrastructure: no business logic, no feature-specific models.

**Rule:** `core/payments/` is a facade layer. It wraps `flutter-payments` implementations and is the ONLY entry point for any payment operation from the presentation layer.

---

## gRPC Layer Placement

| Artifact | Location |
|---|---|
| gRPC client | `lib/core/network/grpc_client.dart` |
| Generated proto stubs | `lib/core/grpc/generated/` (committed to git, regenerated from `proto/`) |
| Proto source files | `proto/services/*.proto` (at repo root, not inside `lib/`) |
| gRPC data sources | `lib/features/<feature>/data/datasources/<feature>_grpc_datasource.dart` |

**Rule:** The domain repository interface is transport-agnostic: it must never reference gRPC or Dio types. The interface accepts and returns domain entities only.

**Rule:** `lib/core/grpc/generated/` must be committed. Proto stubs are generated artifacts but are part of the build contract. Regenerate with:

```bash
protoc --dart_out=grpc:lib/core/grpc/generated/ -I proto proto/services/*.proto
```

---

## Adaptive Layout Rules

| Screen width | Layout | Navigation pattern |
|---|---|---|
| < 600px (phone) | Single pane | `NavigationBar` (bottom) or `CupertinoTabBar` |
| 600px–1024px (tablet / foldable unfolded) | Optional dual pane | `NavigationRail` (left) |
| > 1024px (desktop / large tablet) | Dual pane required | `NavigationDrawer` (left) |

**Rule:** Use `LayoutBuilder` at the root shell scaffold to select the layout: never `MediaQuery.size` inside feature screens.

**Rule:** Foldable hinge detection must use `MediaQuery.displayFeaturesOf(context)` and check for `DisplayFeatureType.fold` or `DisplayFeatureType.hinge`. Never hard-code pixel breakpoints for foldable detection.

```dart
// lib/core/layout/fold.dart
import 'dart:ui' show DisplayFeatureType;

import 'package:material_ui/material_ui.dart';

// Correct foldable detection
bool hasFold(BuildContext context) {
  final displayFeatures = MediaQuery.displayFeaturesOf(context);
  return displayFeatures.any(
    (f) => f.type == DisplayFeatureType.fold || f.type == DisplayFeatureType.hinge,
  );
}
```

---

## Payment Layer Placement

`features/payments/` is a full Clean Architecture feature with all three layers:

| Layer | Contents |
|---|---|
| `domain/` | `PaymentRepository` interface, `Purchase` entity, `SubscriptionTier` entity, use cases |
| `data/` | `StripeDataSource`, `RevenueCatDataSource`, `ApplePayDataSource` |
| `presentation/` | `PaywallPage`, `PurchaseConfirmationPage`, `SubscriptionManagementPage` |

**Rule:** `core/payments/PaymentService` is a facade consumed by other features (e.g., premium feature gate checks). No feature outside `features/payments/` imports from `features/payments/data/` directly.

**Rule:** Receipt validation must happen server-side. The payment data source calls your backend, which validates the receipt with Apple or Google. Never trust receipt data validated client-side only.
