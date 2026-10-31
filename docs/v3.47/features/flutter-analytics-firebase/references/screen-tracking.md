---
title: "Firebase Analytics screen tracking with go_router"
author: MOZGII, Inc.
lang: en
tags: [flutter, firebase, analytics, go-router]
description: A complete go_router setup with one FirebaseAnalyticsObserver, named routes and tab switch logging.
---
# Screen Tracking with go_router

`FirebaseAnalyticsObserver` comes from `package:firebase_analytics/observer.dart`. One observer on the `GoRouter` is enough: go_router adds the router's observers to each shell branch's navigator, unless a shell sets `notifyRootObserver: false`. A second instance in a branch's `observers` list would count that branch's screens twice. Switching tabs moves between navigators without a push, so the tab bar logs that itself.

```dart
// lib/core/router/router.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_analytics/observer.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final analytics = FirebaseAnalytics.instance;
  final router = GoRouter(
    initialLocation: '/home',
    observers: [FirebaseAnalyticsObserver(analytics: analytics)],
    routes: [
      StatefulShellRoute.indexedStack(
        builder: (context, state, shell) => Scaffold(
          body: shell,
          bottomNavigationBar: NavigationBar(
            selectedIndex: shell.currentIndex,
            onDestinationSelected: (index) {
              shell.goBranch(index);
              // Switching tabs pushes nothing, so log it here.
              analytics.logScreenView(
                screenName: index == 0 ? 'home' : 'settings',
              );
            },
            destinations: const [
              NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
              NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
            ],
          ),
        ),
        branches: [
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/home',
                name: 'home',
                builder: (context, state) => const Placeholder(),
              ),
            ],
          ),
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/settings',
                name: 'settings',
                builder: (context, state) => const Placeholder(),
              ),
            ],
          ),
        ],
      ),
    ],
  );
  ref.onDispose(router.dispose);
  return router;
});
```
