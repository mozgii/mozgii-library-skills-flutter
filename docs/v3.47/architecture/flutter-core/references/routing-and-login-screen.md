---
title: Typed routes, the auth guard and the login screen
author: MOZGII, Inc.
lang: en
tags: [flutter, go_router, auth, routing]
description: Typed go_router routes, a redirect guard that survives auth changes without rebuilding the router, and a login screen with loading, error and accessibility states.
---
# Typed Routes, the Auth Guard and the Login Screen

The router is a singleton that reads auth state when it needs to. It never rebuilds. This file holds the three pieces that make that work.

## Route Definitions With go_router_builder

Each screen is a class. `go_router_builder` 4 generates the `$HomeRoute` mixin and `$appRoutes`, so a missing parameter is a compile error rather than a runtime 404.

```dart
// lib/core/router/routes.dart
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';

import '../../features/auth/presentation/pages/login_page.dart';

part 'routes.g.dart';

@TypedGoRoute<HomeRoute>(path: '/')
class HomeRoute extends GoRouteData with $HomeRoute {
  const HomeRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) =>
      const Scaffold(body: Center(child: Text('Home')));
}

@TypedGoRoute<ProfileRoute>(path: '/profile/:userId')
class ProfileRoute extends GoRouteData with $ProfileRoute {
  const ProfileRoute({required this.userId});
  final String userId;

  @override
  Widget build(BuildContext context, GoRouterState state) =>
      Scaffold(body: Center(child: Text('Profile $userId')));
}

@TypedGoRoute<LoginRoute>(path: '/auth/login')
class LoginRoute extends GoRouteData with $LoginRoute {
  const LoginRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) => const LoginPage();
}
```

## An Auth Guard That Never Rebuilds GoRouter

The router provider must not `ref.watch` auth. A watch rebuilds the provider on every auth change, which makes a new `GoRouter`, which throws away the whole navigation stack.

Instead the provider is `keepAlive`, builds the router once, and uses `ref.listen` to tick a `ValueNotifier` passed as `refreshListenable`. On each tick, `go_router` runs `redirect` again against the live stack, and `redirect` uses `ref.read` to get the current state.

```dart
// lib/core/router/router.dart
import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

import '../../features/auth/presentation/providers/auth_provider.dart';
import 'routes.dart';

part 'router.g.dart';

/// True for /auth and anything under it, never for /authors.
bool isAuthLocation(String location) =>
    location == '/auth' || location.startsWith('/auth/');

@Riverpod(keepAlive: true)
GoRouter router(Ref ref) {
  final authChanges = ValueNotifier<int>(0);
  ref.listen(authProvider, (_, _) => authChanges.value++);

  final router = GoRouter(
    initialLocation: '/',
    refreshListenable: authChanges,
    redirect: (context, state) {
      final isAuth = ref.read(authProvider) is AuthStateAuthenticated;
      final isOnAuth = isAuthLocation(state.matchedLocation);
      if (!isAuth && !isOnAuth) return '/auth/login';
      if (isAuth && isOnAuth) return '/';
      return null;
    },
    routes: $appRoutes,
  );

  ref.onDispose(() {
    router.dispose();
    authChanges.dispose();
  });
  return router;
}
```

Connect it once at the top of the app. `routerConfig` takes the same object on every build, because the provider never changes.

```dart
// lib/app.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import 'core/router/router.dart';

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) => MaterialApp.router(
    routerConfig: ref.watch(routerProvider),
    localizationsDelegates: GlobalMaterialLocalizations.delegates,
  );
}
```

## Login Screen With Loading, Error and Accessibility States

The screen holds no logic. It watches the auth state for the loading flag, uses `ref.listen` for the error snackbar, and calls the notifier. The redirect does the navigation once sign-in succeeds, so this widget never navigates. `AutofillGroup` and `autofillHints` let password managers fill the form, and the error text sits in a live region so a screen reader announces it.

```dart
// lib/features/auth/presentation/pages/login_page.dart
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../providers/auth_provider.dart';

class LoginPage extends HookConsumerWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final email = useTextEditingController();
    final password = useTextEditingController();
    final formKey = useMemoized(GlobalKey<FormState>.new);
    final auth = ref.watch(authProvider);
    final isLoading = auth is AuthStateLoading;

    ref.listen(authProvider, (_, next) {
      if (next is AuthStateError) {
        ScaffoldMessenger.of(context)
          ..hideCurrentSnackBar()
          ..showSnackBar(SnackBar(content: Text(next.message)));
      }
    });

    void submit() {
      if (formKey.currentState?.validate() ?? false) {
        ref
            .read(authProvider.notifier)
            .signIn(email.text.trim(), password.text);
      }
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Sign in')),
      body: SafeArea(
        child: AutofillGroup(
          child: Form(
            key: formKey,
            child: ListView(
              padding: const EdgeInsets.all(24),
              children: [
                TextFormField(
                  controller: email,
                  keyboardType: TextInputType.emailAddress,
                  autofillHints: const [AutofillHints.email],
                  textInputAction: TextInputAction.next,
                  decoration: const InputDecoration(labelText: 'Email'),
                  validator: (v) => (v == null || !v.contains('@'))
                      ? 'Enter your email'
                      : null,
                ),
                const SizedBox(height: 16),
                TextFormField(
                  controller: password,
                  obscureText: true,
                  autofillHints: const [AutofillHints.password],
                  onFieldSubmitted: (_) => submit(),
                  decoration: const InputDecoration(labelText: 'Password'),
                  validator: (v) =>
                      (v == null || v.isEmpty) ? 'Enter your password' : null,
                ),
                const SizedBox(height: 24),
                FilledButton(
                  onPressed: isLoading ? null : submit,
                  child: isLoading
                      ? const SizedBox.square(
                          dimension: 20,
                          child: CircularProgressIndicator(
                            strokeWidth: 2,
                            semanticsLabel: 'Signing in',
                          ),
                        )
                      : const Text('Sign in'),
                ),
                if (auth is AuthStateError)
                  Semantics(
                    liveRegion: true,
                    child: Padding(
                      padding: const EdgeInsets.only(top: 16),
                      child: Text(
                        authLabel(auth),
                        style: TextStyle(
                          color: Theme.of(context).colorScheme.error,
                        ),
                      ),
                    ),
                  ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

## Reading the Current Location in a Redirect

`state.matchedLocation` is the location matched so far with its parameters filled in, not the route's pattern: `/family/f2` for `GoRoute('/family/:id')`, and with no query string. That makes it what a guard compares against. To match one group of public routes, compare whole path segments, `location == '/auth' || location.startsWith('/auth/')`: a bare `startsWith('/auth')` also matches `/authors`, which lets signed-out people in and bounces signed-in ones. `redirect` runs again on each `refreshListenable` tick and on every navigation, so it has to stay synchronous and cheap. `redirectLimit` defaults to 5, and two redirects that point at each other hit it with an error page.
