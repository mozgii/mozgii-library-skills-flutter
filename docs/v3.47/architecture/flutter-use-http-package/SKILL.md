---
name: flutter-use-http-package
description: Calls REST APIs from Flutter with the http package - GET, POST, PUT and DELETE, headers and auth, timeouts, retries, status handling, background JSON parsing, native clients, platform permissions and offline tests with MockClient. Use when asked to fetch from an API, send data to a server, add an API client, handle a failed request, or fix a network call that works on one platform only.
metadata:
  tags:
  - flutter
  - networking
  - rest
---
# REST Calls With the http Package

The `http` package is the Dart team's small, composable client. It suits apps with a handful of endpoints, and its `Client` interface makes every call testable offline. This skill builds a client that treats every failure as a value the UI can show, rather than a spinner that never stops.

## Step 0: Check Versions and Permissions

```bash
flutter --version
flutter pub add http
grep -n "INTERNET" android/app/src/main/AndroidManifest.xml
grep -n "network.client" macos/Runner/*.entitlements
```

The examples were compiled, not run on a device, so make one real request from a release build on each platform you ship. Debug Android builds get internet access from a debug manifest, so a missing permission only shows in release. Add it to the main manifest:

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
```

macOS needs the client entitlement in both `macos/Runner/DebugProfile.entitlements` and `macos/Runner/Release.entitlements`:

```xml
<key>com.apple.security.network.client</key>
<true/>
```

The examples also import `material_ui: ^1.9.0`.

## Choosing a Client

| Need | Choose | Why the others lose here |
|---|---|---|
| A few REST calls, minimal dependencies | `http` | `dio` brings features you won't use |
| Interceptors, multipart progress, cancel tokens | `dio` (`flutter-network`) | `http` would need that written by hand |
| Native proxy and VPN support, HTTP/3 | `cupertino_http` on Apple, `cronet_http` on Android, behind `http.Client` | `dart:io` ignores the system proxy |
| Retry on failure | `RetryClient` from `package:http/retry.dart` | A hand-written loop tends to retry POSTs unsafely |

## Request Rules

Build URLs with `Uri.https('api.example.com', '/v1/expenses', {'month': '2026-10'})`, which encodes query values for you. Keep one `http.Client` for the app's lifetime and close it on shutdown, because a client reuses connections and a one-off call doesn't. Every request gets a `.timeout`, since the package sets none and a stalled socket otherwise waits forever.

For a write, encode the body with `jsonEncode` and set `content-type: application/json`. Treat a status from 200 to 299 as success, since a create returns 201 and a delete often returns 204 with an empty body. Throw on anything else. Returning `null` on failure leaves a `FutureBuilder` loading forever, because it never reaches its error state.

## Worked Example: An Expenses API Client

Failures become one sealed exception, so the UI's `switch` can show the right message for each.

```dart
// lib/data/expense_client.dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:isolate';

import 'package:http/http.dart' as http;

import 'expense.dart';

sealed class ApiFailure implements Exception {
  const ApiFailure();
}

final class Offline extends ApiFailure {
  const Offline();
}

final class TimedOut extends ApiFailure {
  const TimedOut();
}

final class ServerRejected extends ApiFailure {
  const ServerRejected(this.status, this.body);
  final int status;
  final String body;
}

class ExpenseClient {
  ExpenseClient(this._http, {required this.token});

  static const _host = 'api.example.com';
  static const _timeout = Duration(seconds: 15);

  final http.Client _http;
  final Future<String> Function() token;

  Future<Map<String, String>> _headers() async => {
        HttpHeaders.authorizationHeader: 'Bearer ${await token()}',
        HttpHeaders.acceptHeader: 'application/json',
        HttpHeaders.contentTypeHeader: 'application/json',
      };

  Future<http.Response> _send(Future<http.Response> Function() call) async {
    try {
      final response = await call().timeout(_timeout);
      if (response.statusCode >= 200 && response.statusCode < 300) return response;
      throw ServerRejected(response.statusCode, response.body);
    } on TimeoutException {
      throw const TimedOut();
    } on SocketException {
      throw const Offline();
    } on http.ClientException {
      throw const Offline();
    }
  }

  Future<List<Expense>> list({required String month}) async {
    final url = Uri.https(_host, '/v1/expenses', {'month': month});
    final headers = await _headers();
    final response = await _send(() => _http.get(url, headers: headers));
    final body = response.body;
    // Large payloads parse in a background isolate, off the UI thread.
    return Isolate.run(() => [
          for (final row in jsonDecode(body) as List<dynamic>)
            Expense.fromJson(row as Map<String, dynamic>),
        ]);
  }

  Future<Expense> create(NewExpense draft) async {
    final url = Uri.https(_host, '/v1/expenses');
    final headers = await _headers();
    final response = await _send(
      () => _http.post(url, headers: headers, body: jsonEncode(draft.toJson())),
    );
    return Expense.fromJson(jsonDecode(response.body) as Map<String, dynamic>);
  }

  Future<void> delete(String id) async {
    final url = Uri.https(_host, '/v1/expenses/$id');
    final headers = await _headers();
    await _send(() => _http.delete(url, headers: headers));
  }
}
```

`lib/data/expense.dart` holds `Expense` and `NewExpense`, plain models with `fromJson` and `toJson` (`flutter-implement-json-serialization`).

## Showing It in the UI

`expense.dart` holds the two models the client reads and writes:

```dart
// lib/data/expense.dart
class Expense {
  const Expense({required this.id, required this.note});

  factory Expense.fromJson(Map<String, dynamic> json) =>
      Expense(id: json['id'] as String, note: json['note'] as String);

  final String id;
  final String note;
}

class NewExpense {
  const NewExpense({required this.note});

  final String note;

  Map<String, dynamic> toJson() => {'note': note};
}
```

Store the `Future` in `initState`. Creating it inside `build` starts a new request on every rebuild.

```dart
// lib/ui/expense_list.dart
import 'package:material_ui/material_ui.dart';

import '../data/expense.dart';
import '../data/expense_client.dart';

class ExpenseList extends StatefulWidget {
  const ExpenseList({super.key, required this.client});

  final ExpenseClient client;

  @override
  State<ExpenseList> createState() => _ExpenseListState();
}

class _ExpenseListState extends State<ExpenseList> {
  late Future<List<Expense>> _expenses = widget.client.list(month: '2026-10');

  void _retry() => setState(() {
        _expenses = widget.client.list(month: '2026-10');
      });

  @override
  Widget build(BuildContext context) => FutureBuilder<List<Expense>>(
        future: _expenses,
        builder: (context, snapshot) {
          if (snapshot.hasError) {
            final message = switch (snapshot.error) {
              Offline() => 'You are offline',
              TimedOut() => 'The server took too long',
              ServerRejected(status: 401) => 'Please sign in again',
              _ => 'Something went wrong',
            };
            return Center(child: TextButton(onPressed: _retry, child: Text(message)));
          }
          if (!snapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }
          final expenses = snapshot.requireData;
          if (expenses.isEmpty) return const Center(child: Text('No expenses yet'));
          return ListView.builder(
            itemCount: expenses.length,
            itemBuilder: (context, i) => ListTile(title: Text(expenses[i].note)),
          );
        },
      );
}
```

## Retries and Parsing in the Background

`RetryClient` wraps any client and retries on 503 by default. Pass it only to reads, because a retried POST can create a duplicate. It backs off exponentially.

```dart
import 'package:http/http.dart' as http;
import 'package:http/retry.dart';

final reads = RetryClient(http.Client(), retries: 3);
```

For parsing, `Isolate.run` moves the work off the UI thread, and it accepts a closure. `compute(parseExpenses, body)` from `package:flutter/foundation.dart` also works and needs a top-level or static function; use it for code that must also run on web, where `Isolate.run` isn't available.

## Verify

`MockClient` from `package:http/testing.dart` answers requests with no network at all.

```dart
// test/data/expense_client_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:http/http.dart' as http;
import 'package:http/testing.dart';
import 'package:yourapp/data/expense_client.dart';

void main() {
  ExpenseClient clientReturning(int status, String body) => ExpenseClient(
        MockClient((request) async {
          expect(request.headers['authorization'], 'Bearer t');
          return http.Response(body, status);
        }),
        token: () async => 't',
      );

  test('a 200 list parses', () async {
    final client = clientReturning(200, '[{"id":"e1","note":"Lunch"}]');
    expect((await client.list(month: '2026-10')).single.note, 'Lunch');
  });

  test('a 401 becomes ServerRejected', () {
    expect(
      clientReturning(401, '').list(month: '2026-10'),
      throwsA(isA<ServerRejected>().having((f) => f.status, 'status', 401)),
    );
  });
}
```

```bash
flutter test test/data/expense_client_test.dart
flutter build apk --release   # confirms the release manifest has INTERNET
```

## Gotchas

1. **`SocketException: Failed host lookup` in release only.** Symptom: it works in debug on Android. Cause: `INTERNET` is missing from the main manifest. Fix: add the permission.
2. **`Operation not permitted` on macOS.** Symptom: every request fails. Cause: the network client entitlement is missing. Fix: add it to both entitlement files.
3. **The spinner never stops.** Symptom: an endless loading state. Cause: the function returned `null`, or there's no timeout. Fix: throw on failure, and add `.timeout`.
4. **The request repeats on each rebuild.** Symptom: the server log shows bursts. Cause: the `Future` was created in `build`. Fix: store it in `initState`.
5. **Garbled accented text.** Symptom: `Ã©` instead of `é`. Cause: `response.body` guessed latin1 because the server sent no charset. Fix: decode with `utf8.decode(response.bodyBytes)`.
6. **Cleartext traffic blocked.** Symptom: `Cleartext HTTP traffic not permitted`, or an App Transport Security error. Cause: an `http://` URL. Fix: use HTTPS.

## Anti-Patterns

- Calling `http.get` from a widget. Put it in a client class behind an interface.
- Creating a new `Client` per request. Share one.
- Putting API keys in the app. Keep them on a backend (`flutter-security`).
- Catching every exception and returning an empty list. Users then see "no data" for an outage.

## Pre-Merge Checklist

- [ ] Every request has a timeout
- [ ] Non-2xx responses throw a typed failure
- [ ] Loading, empty, error and success states all render
- [ ] Android and macOS permissions are in place
- [ ] Tests use `MockClient`, with no real network

## See Also

- `flutter-network` for `dio`, gRPC, GraphQL and caching
- `flutter-implement-json-serialization` for the models
- `flutter-security` for keys and certificate pinning
