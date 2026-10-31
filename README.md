# Flutter skills

A library of Agent Skills for building production Flutter and Dart apps, for Claude to search with a citation per
passage.

- `docs/v3.47/<group>/<skill>/SKILL.md`: 35 skills. 26 cover Flutter by area (architecture, state, navigation, data,
  networking, auth, security, payments, notifications, AI, design, animation, performance, testing, agent tooling),
  and 9 cover Dart language and tooling (patterns, primary constructors, unit tests, analysis, dependencies, native
  assets, documentation, paths, command-line tools)
- `docs/v3.47/rules/<rule>/RULE.md`: 5 rules the skills assume (architecture, code quality, Firebase, store compliance, hot
  reload)
- `library.json`: name `flutter`, title, version, the self-test minimum
- `questions.json`: the self-test; each question names a phrase its answering passage holds
- `versions.json`: every pub.dev package the library names, with the version it teaches and the date checked. A package
  taught below its latest has `held`, saying why
- `sources.json`: every source read as context, at the commit or version read

The built pack lands in `dist/` (not in git). Each release is tagged, with `dist/pack.tar.gz` as its release asset.

## Versions

The library targets Flutter 3.47.9 and Dart 3.13.7, the stable releases on 2026-10-30. Every skill's examples import
`material_ui`, and each skill that uses it says how a `flutter create` project migrates; `flutter-core`, Step 0, has
the detail. The SDK versions aren't
in `versions.json`, because neither SDK publishes a usable latest GitHub release for the drift check to read; compare
them with `flutter --version`.

Every pin in the library resolves together on those SDKs. Two sit below their latest: `build_runner: ^2.16.1` and
`connectivity_plus: ^7.3.1`, each with its reason in `held` in `versions.json`.

## Sources

Every text in this library is our own writing. The sources below were read as context for facts and topics, and
nothing was copied from them. `sources.json` holds the exact commit or version of each, and which documents it
informed.

| Publisher | Source | Licence | What it informed |
|---|---|---|---|
| Flutter team | [flutter/agent-plugins](https://github.com/flutter/agent-plugins) (`flutter/skills` redirects here) | BSD-3-Clause | Which topics a Flutter developer needs; the 10 `flutter-*` task skills and the hot reload rule |
| Dart team | [dart-lang/skills](https://github.com/dart-lang/skills) | BSD-3-Clause | The Dart skills |
| Dart team | [dart-lang/ai](https://github.com/dart-lang/ai) | BSD-3-Clause | MCP server tools, categories and flags |
| Flutter team | [flutter/packages](https://github.com/flutter/packages) | BSD-3-Clause | The `material_ui` and `cupertino_ui` move |
| Flutter team | [docs.flutter.dev](https://docs.flutter.dev) | CC BY 4.0 (text), BSD-3-Clause (code) | Breaking changes, Impeller, localization, previews, architecture weights, agent setup |
| Dart team | [dart.dev](https://dart.dev) | CC BY 4.0 (text), BSD-3-Clause (code) | Build hooks, package skills |
| Google | Play Console Help, Android developer docs, Firebase docs | Facts only | Target API, service fees, payments and user data policy, purchase verification, consent and collection keys, App Check, FCM tokens, Firestore's offline cache, exact alarms, the notification permission |
| Apple | App Review Guidelines, App Store programs, Apple Developer documentation, the UIKit header in Xcode 27.2 | Facts only | Login and purchase rules, commission rates, privacy manifest keys, App Store Server API, screenshot sizes, the notification prompt, Reduce Motion, `UIGlassEffect` |
| Flutter team, Dart team | [Engine and `flutter_tools` source](https://github.com/flutter/flutter), [Dart SDK source](https://github.com/dart-lang/sdk) | BSD-3-Clause | Reduce Motion on each platform, build flag errors, isolates |
| Supabase | Realtime documentation | Apache-2.0 | Row level security on Postgres changes |
| Firebase | [firebase/agent-skills](https://github.com/firebase/agent-skills), [firebase-tools](https://github.com/firebase/firebase-tools) | Apache-2.0, MIT | AI Logic setup, which the Firebase skills and rule take as given |
| Stripe | [stripe/ai](https://github.com/stripe/ai) `stripe-best-practices` | MIT | Webhooks as a required step, key handling in mobile apps |
| Supabase | [supabase/agent-skills](https://github.com/supabase/agent-skills) | MIT | Row level security and client key rules |
| gRPC authors | [grpc/grpc-web](https://github.com/grpc/grpc-web) | Apache-2.0 | The proxy gRPC-Web needs |
| OWASP | Mobile Top 10 and the Password Storage Cheat Sheet | CC BY-SA 4.0 | Risk names and Argon2id parameters |
| Stripe, RevenueCat | Their developer docs | Facts only | Webhook events and delivery behavior |
| Package authors | Changelogs, READMEs and API sources of pub.dev packages | Each package's own licence, listed in `NOTICE` | Breaking changes, migrations and current API names |
