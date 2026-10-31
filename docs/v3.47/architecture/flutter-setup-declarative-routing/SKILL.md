---
name: flutter-setup-declarative-routing
description: Sets up go_router in a Flutter app - a route tree, path and query parameters, auth redirects, persistent tabs with StatefulShellRoute, clean web URLs, and Android App Links plus iOS Universal Links with their verification files. Use when asked to add navigation, deep links, a bottom bar that keeps its tab state, a login redirect, or to open the app from a URL.
metadata:
  tags:
  - flutter
  - navigation
  - go-router
  - deep-linking
---
# Declarative Routing With go_router

Declarative routing makes the URL the source of truth: a location string decides which screens are on the stack. That gives you deep links, browser history and redirects from one route table instead of scattered `Navigator` calls.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(go_router|go_router_builder|app_links|material_ui):" pubspec.yaml
flutter pub add go_router
```

`go_router` 18 imports `material_ui`, which needs Flutter 3.44 or later. Migrate first with `dart fix --apply --code=migrate_design_widgets`, or stay on the 17 line. Since version 15, paths are case sensitive, so `/Home` no longer matches `/home`. The examples also import `material_ui: ^1.9.0`.

## Choosing Navigation

| Need | Choose | Why the others lose here |
|---|---|---|
| A short stack, no URLs | `Navigator.push` | `go_router` adds setup you won't use |
| Deep links, web, redirects | `go_router` | Hand-written `Router` delegates are long and easy to break |
| The same, with compile-time safety | `go_router` + `go_router_builder` | String paths can drift |
| Tabs that keep their stacks | `StatefulShellRoute.indexedStack` | `ShellRoute` drops branch state |
| A persistent frame, no per-tab state | `ShellRoute` | The stateful variant keeps stacks you don't need |

## go or push

`context.go('/expenses/42')` replaces the stack with what the route tree says that location is. `context.push('/expenses/42')` puts a screen on top of the current one, and the result can be returned through `context.pop(value)`. Use `go` for moving between sections and `push` for a detail you'll come back from. `context.goNamed('expense', pathParameters: {'id': '42'})` navigates by name, so paths can change freely.

## Worked Example: Expenses With Tabs, Auth and Deep Links

```dart
// lib/app_router.dart
import 'package:go_router/go_router.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/auth_controller.dart';
import 'package:yourapp/safe_return_path.dart';
import 'package:yourapp/screens.dart';

final _rootKey = GlobalKey<NavigatorState>();

GoRouter buildRouter(AuthController auth) => GoRouter(
      navigatorKey: _rootKey,
      initialLocation: '/expenses',
      refreshListenable: auth, // re-runs redirect when sign-in changes
      redirect: (context, state) {
        final signingIn = state.matchedLocation == '/sign-in';
        if (!auth.isSignedIn) {
          // Keep the deep link, so sign-in can return to it.
          return signingIn
              ? null
              : Uri(path: '/sign-in', queryParameters: {'from': state.uri.toString()}).toString();
        }
        if (signingIn) {
          return safeReturnPath(state.uri.queryParameters['from']);
        }
        return null;
      },
      errorBuilder: (context, state) => NotFoundScreen(location: state.uri.toString()),
      routes: [
        GoRoute(path: '/sign-in', builder: (context, state) => const SignInScreen()),
        StatefulShellRoute.indexedStack(
          builder: (context, state, shell) => AppFrame(shell: shell),
          branches: [
            StatefulShellBranch(routes: [
              GoRoute(
                path: '/expenses',
                builder: (context, state) => const ExpenseListScreen(),
                routes: [
                  GoRoute(
                    path: ':id',
                    name: 'expense',
                    builder: (context, state) => ExpenseDetailScreen(
                      id: state.pathParameters['id']!,
                      highlight: state.uri.queryParameters['highlight'],
                    ),
                  ),
                ],
              ),
            ]),
            StatefulShellBranch(routes: [
              GoRoute(path: '/budget', builder: (context, state) => const BudgetScreen()),
            ]),
          ],
        ),
      ],
    );

class AppFrame extends StatelessWidget {
  const AppFrame({super.key, required this.shell});

  final StatefulNavigationShell shell;

  @override
  Widget build(BuildContext context) => Scaffold(
        body: shell,
        bottomNavigationBar: NavigationBar(
          selectedIndex: shell.currentIndex,
          // Tapping the active tab returns to that branch's first route.
          onDestinationSelected: (index) =>
              shell.goBranch(index, initialLocation: index == shell.currentIndex),
          destinations: const [
            NavigationDestination(icon: Icon(Icons.receipt_long), label: 'Expenses'),
            NavigationDestination(icon: Icon(Icons.pie_chart), label: 'Budget'),
          ],
        ),
      );
}
```

`from` comes from a URL, so anyone can craft it. `startsWith('/')` isn't enough: `/\t/evil.example` and `/\n/evil.example` pass that test, and URL parsers strip tabs and newlines, which leaves `//evil.example`. A backslash can do the same. The guard refuses control characters and backslashes, in raw and percent-encoded form, before it checks the shape.

```dart
// lib/safe_return_path.dart
final _unsafe = RegExp(r'[\\\x00-\x1F\x7F]');

/// Returns [from] only when it's a path inside this app.
String safeReturnPath(String? from, {String fallback = '/expenses'}) {
  if (from == null || from.isEmpty) return fallback;
  final String decoded;
  try {
    decoded = Uri.decodeFull(from);
  } on ArgumentError {
    return fallback;
  }
  if (_unsafe.hasMatch(from) || _unsafe.hasMatch(decoded)) return fallback;
  if (!decoded.startsWith('/') || decoded.startsWith('//')) return fallback;
  final uri = Uri.tryParse(from);
  if (uri == null || uri.hasScheme || uri.hasAuthority) return fallback;
  return from;
}
```

```dart
// lib/auth_controller.dart
import 'package:flutter/foundation.dart';

class AuthController extends ChangeNotifier {
  bool _signedIn = false;

  bool get isSignedIn => _signedIn;

  void setSignedIn(bool value) {
    _signedIn = value;
    notifyListeners();
  }
}
```

```dart
// lib/main.dart
import 'package:flutter_web_plugins/url_strategy.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/app_router.dart';
import 'package:yourapp/auth_controller.dart';

void main() {
  usePathUrlStrategy(); // web: /expenses/42 instead of /#/expenses/42
  runApp(const ExpenseApp());
}

class ExpenseApp extends StatefulWidget {
  const ExpenseApp({super.key});

  @override
  State<ExpenseApp> createState() => _ExpenseAppState();
}

class _ExpenseAppState extends State<ExpenseApp> {
  final _auth = AuthController();
  late final _router = buildRouter(_auth); // created once: never inside build

  @override
  void dispose() {
    _router.dispose();
    _auth.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => MaterialApp.router(routerConfig: _router);
}
```

`usePathUrlStrategy` lives in `flutter_web_plugins`, an SDK package you add with `flutter pub add flutter_web_plugins --sdk=flutter`. Path URLs need a web server that sends every path to `index.html`.

The screens are ordinary widgets; `references/example-screens.md` has stand-ins that let this example run.

## Platform Deep Linking Setup

The route table handles a link once it reaches the app. Each platform also has to agree that your app owns the domain. Flutter's own deep link handler has been on by default since Flutter 3.27, so most apps need only the steps below.

### Android App Links

Add an intent filter to the main activity in `android/app/src/main/AndroidManifest.xml`. `autoVerify` makes Android check your domain at install time.

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="expenses.example.com" />
</intent-filter>
```

Serve this at `https://expenses.example.com/.well-known/assetlinks.json`, using the signing certificate's SHA-256 fingerprint. With Play App Signing, take the fingerprint from the Play Console, not from your upload key.

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.expenses",
    "sha256_cert_fingerprints": ["AA:BB:CC:00:00:00"]
  }
}]
```

### iOS Universal Links

Add the Associated Domains capability, which writes this into `ios/Runner/Runner.entitlements`:

```xml
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:expenses.example.com</string>
</array>
```

Serve this with no file extension at `https://expenses.example.com/.well-known/apple-app-site-association`, over HTTPS, as `application/json`:

```json
{
  "applinks": {
    "details": [{
      "appIDs": ["TEAMID.com.example.expenses"],
      "components": [{ "/": "/expenses/*" }]
    }]
  }
}
```

### Using a Third-Party Link Plugin

If you use `app_links` or a similar plugin, switch Flutter's handler off so the two don't both process a link. On iOS, set `FlutterDeepLinkingEnabled` to `false` in `Info.plist`. On Android, add this inside the `<activity>`:

```xml
<meta-data android:name="flutter_deeplinking_enabled" android:value="false" />
```

## Verify

The examples were compiled, not run on a device, so open your deep links on real hardware. Send a real link to a running app on each platform, in both the cold and the warm state:

```bash
adb shell 'am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "https://expenses.example.com/expenses/42"' com.example.expenses
xcrun simctl openurl booted https://expenses.example.com/expenses/42
adb shell pm get-app-links com.example.expenses     # expect "verified" for the domain
curl -I https://expenses.example.com/.well-known/assetlinks.json
```

Test the redirect's open-redirect guard with attack cases. Each of these must fall back, so a crafted link never leaves the app:

```dart
// test/safe_return_path_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/safe_return_path.dart';

void main() {
  const attacks = [
    '//evil.example',
    '/\t/evil.example',
    '/\n/evil.example',
    r'/\evil.example',
    r'\\evil.example',
    'https://evil.example',
    'javascript:alert(1)',
    '/%09/evil.example',
    '/%0a/evil.example',
    '/%5Cevil.example',
    '%2F%2Fevil.example',
    '/%zz',
    'expenses',
    '',
  ];

  for (final attack in attacks) {
    test('refuses ${Uri.encodeComponent(attack)}', () {
      expect(safeReturnPath(attack), '/expenses');
    });
  }

  test('refuses a missing value', () {
    expect(safeReturnPath(null), '/expenses');
  });

  test('keeps a path inside the app, with its query', () {
    expect(safeReturnPath('/expenses/42?tab=a'), '/expenses/42?tab=a');
  });
}
```

```dart
// test/app_router_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:material_ui/material_ui.dart';
import 'package:yourapp/app_router.dart';
import 'package:yourapp/auth_controller.dart';

void main() {
  testWidgets('a signed-out deep link keeps its target', (tester) async {
    final router = buildRouter(AuthController());
    addTearDown(router.dispose);
    await tester.pumpWidget(MaterialApp.router(routerConfig: router));

    router.go('/expenses/42');
    await tester.pumpAndSettle();

    expect(router.state.uri.path, '/sign-in');
    expect(router.state.uri.queryParameters['from'], '/expenses/42');
  });

  testWidgets('signing in with a hostile target stays in the app', (tester) async {
    final auth = AuthController();
    final router = buildRouter(auth);
    addTearDown(router.dispose);
    await tester.pumpWidget(MaterialApp.router(routerConfig: router));

    router.go('/sign-in?from=${Uri.encodeComponent('/\t/evil.example')}');
    await tester.pumpAndSettle();
    auth.setSignedIn(true);
    await tester.pumpAndSettle();

    expect(router.state.uri.path, '/expenses');
  });
}
```

## Gotchas

1. **The tab resets on every switch.** Symptom: scroll position and detail pages vanish. Cause: `ShellRoute`, or a `GoRouter` created inside `build`. Fix: use `StatefulShellRoute`, and create the router once.
2. **`Page not found` for a valid URL.** Symptom: the error screen for `/Expenses`. Cause: paths are case sensitive since 15. Fix: use the exact case, or pass `caseSensitive: false`.
3. **The link opens the browser.** Symptom: Android shows Chrome rather than the app. Cause: verification failed. Fix: run `pm get-app-links`, then check the fingerprint and that the file is served without a redirect.
4. **Infinite redirects.** Symptom: `redirect loop detected`. Cause: `redirect` returns a location even when already there. Fix: return `null` once you've arrived.
5. **`Duplicate GlobalKey detected in widget tree` after a redirect.** Symptom: a red error screen. Cause: two navigators share one key. Fix: give each shell its own `GlobalKey<NavigatorState>`.
6. **`context.pop` closes the app.** Symptom: back from a deep link exits. Cause: `go` left only one page. Fix: nest the detail under its list, so the tree supplies a parent.

## Anti-Patterns

- Building the `GoRouter` in `build`. It resets navigation on every rebuild.
- Mixing `Navigator.push` with routes that need redirects. The router can't see those pages.
- Passing whole objects through `extra`. A deep link has no `extra`, so pass an id and load the object.
- Redirecting to a raw query value. Pass it through a guard like `safeReturnPath`.

## Pre-Merge Checklist

- [ ] Every screen reachable by link is a route
- [ ] Redirects are guarded against open-redirect values
- [ ] Both verification files return 200, with no redirect
- [ ] Links were tested with the app closed and in the background
- [ ] An error route exists

## See Also

- `flutter-core` for typed routes with `go_router_builder` 4.5
- `flutter-auth` for the sign-in state the redirect reads
- `flutter-notifications` for opening a route from a notification
