---
name: dart-manage-dependencies
description: Keeps Dart and Flutter dependencies healthy - reading dart pub outdated, caret constraints, upgrading across a major, tightening bounds, resolving version solving failures, retracted and discontinued packages, dependency overrides and lockfile enforcement in CI. Use when pub get fails, a package won't upgrade, version solving failed, pubspec.lock conflicts, or someone asks to update dependencies.
metadata:
  tags:
  - dart
  - flutter
  - pub
  - dependencies
---
# Managing Dart Dependencies

Pub allows one version of each package in an app, so every package and all its transitive dependencies must agree on it. That rules out runtime type clashes, but it means one stale package can hold back everything else. This skill reads the resolver's output and moves a project forward without losing control of what changed.

## Step 0: Take Stock

```bash
flutter --version
dart pub outdated
dart pub deps --style=compact | head -30
```

Use `flutter pub` in a Flutter project when you want the SDK's pinned packages accounted for. The commands are otherwise the same. Commit `pubspec.lock` in apps; packages leave it out.

## Reading dart pub outdated

| Column | Means | How to get it |
|---|---|---|
| Current | What `pubspec.lock` has | Already there |
| Upgradable | The newest your constraints allow | `dart pub upgrade` |
| Resolvable | The newest that works with everything, if you edit constraints | Edit `pubspec.yaml`, or `dart pub upgrade --major-versions` |
| Latest | The newest published, ignoring pre-releases | May need other packages to move first |

## Which Move for Which Situation

| Situation | Do | Why the others lose here |
|---|---|---|
| Routine freshening | `dart pub upgrade` | Touching constraints isn't needed |
| Raise lower bounds to what you tested | `dart pub upgrade --tighten` | Old minimums go unverified |
| Move one package across a major | Edit its constraint, then `dart pub upgrade <pkg>` | A blanket upgrade mixes unrelated changes |
| Many majors at once | `dart pub upgrade --major-versions` | Fine for dev tools, risky for runtime packages |
| A retracted version in the lockfile | Remove only that entry, then `dart pub get` | Deleting the whole lockfile upgrades everything |
| Two packages disagree | Upgrade the older one, then read the solver's reason | An override hides the conflict |
| A fix that hasn't been released | `dependency_overrides` with a Git ref, plus a note | Use it briefly: it applies to your app only |

## Constraints

Use caret syntax: `^1.2.3` means anything from 1.2.3 up to, but not including, 2.0.0. For a 0.x package, the caret stops at the next minor: `^0.5.1` means below 0.6.0, so read 0.x changelogs as if each minor could break you. In `dev_dependencies`, set the lower bound to the version you use, which keeps the solver from picking an ancient tool.

## Worked Example: A Version Solving Failure

```text
Because expense_tracker depends on pinput ^7.0.0 which requires SDK version >=3.12.0,
version solving failed.
```

Read the error from the end: it names the two things that can't both be true. Here the SDK is too old for `pinput` 7, and the choices are to upgrade Flutter or to stay on `pinput` 6.

```bash
flutter upgrade                    # option 1: move the SDK
flutter pub add pinput:^6.0.2      # option 2: hold the package
dart pub deps --style=compact | grep pinput   # who else pulls it in
```

Here's a transitive one, where two of your packages pull different majors of a shared package:

```text
Because every version of package_a depends on intl ^0.19.0 and
every version of package_b depends on intl ^0.20.0, ...
```

Upgrade `package_a`. If no release supports the new version, look at its issue tracker, then use a short-lived override and test the code paths it touches:

```yaml
dependency_overrides:
  intl: ^0.20.3   # until package_a releases with intl 0.20: see its issue tracker
```

## Upgrading Across a Major

1. Run `dart pub outdated` and pick one package.
2. Read its changelog from your version to the target.
3. Edit the constraint and run `dart pub upgrade <pkg>`.
4. Run `dart analyze`, then fix each break. `dart fix --apply` handles some of them.
5. Run the tests, commit, then take the next package.

Check what a major needs before you move. `go_router` 18, `pinput` 7, `shimmer` 4, `animations` 3, `google_fonts` 9 and `dynamic_color` 2 import `material_ui`, which needs Flutter 3.44 or later. Migrate the app first with `dart fix --apply --code=migrate_design_widgets`, or add `material_ui` and run `dart fix --apply`. Otherwise, stay on the previous major of those packages.

## Retracted and Discontinued Packages

A retracted version stays usable only for apps already locked to it, and `dart pub outdated` flags it. Open `pubspec.lock`, delete only that package's block, and run `dart pub get`. Don't delete the whole file: that re-resolves your entire graph at once.

A discontinued package shows a `(discontinued)` note and often names a replacement. Don't wait for it to break. Several packages Flutter tutorials still show have moved: `flutter_gemma` to `flutter_edge_ai`, `sqlite3_flutter_libs` into `sqlite3` 3.x, `dio_cache_interceptor_hive_store` to `http_cache_hive_store`.

## Verify

```bash
dart pub get --enforce-lockfile     # CI: fails if the lockfile doesn't match
dart analyze
dart test
dart pub outdated --json --show-all | grep -cE '"(isDiscontinued|isCurrentRetracted)": true'   # expect 0
```

`--enforce-lockfile` makes CI build exactly what you tested. `flutter pub get --enforce-lockfile` is the Flutter form. The last line needs `--show-all`, because without it the JSON leaves out up-to-date packages, and a discontinued package at its final version is always up to date. `dart pub deps` prints neither word, so it can't detect either state.

## Gotchas

1. **`version solving failed` naming the SDK.** Symptom: a package requires a newer Dart. Cause: your Flutter is behind. Fix: upgrade Flutter, or pin an older package.
2. **`Because ... depends on flutter from sdk which depends on collection 1.19.1`.** Symptom: the SDK pins a shared package. Cause: Flutter fixes some transitive versions. Fix: use a package version that accepts the pin, or upgrade Flutter.
3. **`dart pub upgrade` changes nothing.** Symptom: Latest is ahead of Current. Cause: your constraint caps it. Fix: edit the constraint, or use `--major-versions`.
4. **Duplicate entries after a merge.** Symptom: `pubspec.lock` has conflict markers. Cause: two branches moved packages. Fix: take one side of the file, then run `dart pub get`.
5. **An override ships for a year.** Symptom: an old fix masks a real conflict. Cause: nobody removed it. Fix: leave a dated note, and review overrides at each upgrade.
6. **A 0.x bump breaks the build.** Symptom: compile errors after a "minor" update. Cause: in 0.x, a minor can break. Fix: read the changelog.
7. **`build_runner` conflicts with a generator.** Symptom: `version solving failed` naming `build`. Cause: `build_runner` 2.17.1 needs `build` 5, and `riverpod_generator` 4.0.9, `envied_generator` 1.3.10 and `injectable_generator` 3.1.3 stop below it. `injectable_generator` also holds `analyzer` below what 2.16.2 needs. Fix: use `build_runner: ^2.16.1` until they move.
8. **Two plugins disagree on a transitive major.** Symptom: `version solving failed` naming `dbus`. Cause: `connectivity_plus` 7.3.2 needs `dbus` 0.8 and `geolocator` 14.1.2 needs 0.7. Fix: use `connectivity_plus: ^7.3.1`, which lets the solver take the older release.

## Anti-Patterns

- Deleting `pubspec.lock` to fix a conflict. It hides what changed.
- Using `any` for a real dependency. `intl: any` beside `flutter_localizations` is the known exception, since the SDK pins it.
- Upgrading everything in one commit. You won't know what broke.
- Leaving `dependency_overrides` in a published package. Pub ignores them for your users.

## Pre-Merge Checklist

- [ ] `pubspec.lock` is committed for apps
- [ ] Each override has a dated reason
- [ ] `dart pub outdated --json --show-all` reports nothing discontinued or retracted
- [ ] Analyzer and tests pass on the new versions
- [ ] CI uses `--enforce-lockfile`

## See Also

- `flutter-core` for the wider project setup
- `dart-analyze-and-fix` for fixing what an upgrade breaks
