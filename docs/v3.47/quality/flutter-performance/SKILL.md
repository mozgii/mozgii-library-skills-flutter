---
name: flutter-performance
description: Finds and fixes Flutter jank, slow starts, memory growth and oversized downloads - a DevTools profiling workflow, rebuild reduction with select, const and RepaintBoundary, background isolates, image decode limits, lazy lists, deferred components, app size reduction and production frame monitoring. Use when asked to fix jank or dropped frames, speed up cold start, stop a memory leak, shrink the app, run a pre-release performance audit, or build a long or data-heavy list.
metadata:
  tags:
  - flutter
  - performance
  - impeller
  - profiling
  - isolates
  - optimization
---
# Flutter Performance

Performance work in Flutter means keeping every frame inside its budget, starting fast, holding memory flat and shipping a small download. Measure first: most guesses about jank are wrong, and a fix you can't measure may be a fix for nothing.

The targets this skill uses are ours, not platform rules: 60 fps on a mid-range Android phone such as a Samsung A32, 120 fps on a current iPhone Pro, cold start under 3 seconds on a Pixel 6a, and under 20 MB download on Google Play. A frame has 16 ms at 60 Hz and 8 ms at 120 Hz.

## Step 0: Read the Project's Versions

Advice below was checked against Flutter 3.47.9, `material_ui` 1.9.0, `flutter_riverpod` 3.4.3, `cached_network_image` 4.0.6, `firebase_performance` 0.12.1 and `firebase_analytics` 13.0.1. Compare yours:

```bash
flutter --version
grep -nE "^\s+(material_ui|flutter_riverpod|cached_network_image|firebase_performance|firebase_analytics):" pubspec.yaml
```

A project still importing `package:flutter/material.dart` should move with `dart fix --apply --code=migrate_design_widgets`; every example here imports `package:material_ui/material_ui.dart`. The examples also import `cached_network_image: ^4.0.6`, `firebase_core: ^5.1.0`, `flutter_riverpod: ^3.4.3`, `material_ui: ^1.9.0`.

## Which Fix for Which Symptom

| You measured | Do this first | Why the others lose here |
|---|---|---|
| Frames over budget in the build phase | Narrow what rebuilds: `select`, `const`, smaller widgets | `RepaintBoundary` does nothing for build cost |
| Frames over budget in the raster phase | Remove blurs, clips and saveLayers; add `RepaintBoundary` around what repaints often | `const` doesn't reduce raster work |
| UI freezes during one piece of work | Move it off the UI isolate | A smaller widget tree can't hide a 200 ms parse |
| Memory climbs and never falls | Compare heap snapshots, then decode images smaller | A bigger `imageCache` hides the growth |
| Slow first scroll through images | `memCacheWidth` at display size | Disk cache settings don't change decode size |
| Android download too big | `--analyze-size` on one ABI, then cut the largest item | Blind asset compression misses a heavy dependency |

## Impeller Renderer Status per Platform

Impeller precompiles a small fixed set of shaders when the engine is built, so nothing compiles at runtime and first-animation shader jank disappears.

| Platform | Status | Opt-out |
|---|---|---|
| iOS | Only renderer | None |
| Android API 29+ | Default since Flutter 3.27, on Vulkan | `flutter run --no-enable-impeller`, or `io.flutter.embedding.android.EnableImpeller` set to `false` in `AndroidManifest.xml` |
| Android below API 29, or no Vulkan | Falls back to the legacy OpenGL renderer on its own | Not needed |
| macOS | Default since Flutter 3.47 | `FLTEnableImpeller` set to `false` in `Info.plist`; temporary, due for removal |
| Windows, Linux | Default since Flutter 3.47 | Runner code; temporary, due for removal |
| Web | Skia | Not applicable |

Custom fragment shaders work on Impeller: declare them under `flutter.shaders` in `pubspec.yaml` so the build compiles them. If a project looks wrong only on Impeller, check for a leftover opt-out before you blame the renderer.

## DevTools Profiling Workflow

Profile on a real device in profile mode. Debug mode runs unoptimized code and a simulator's GPU isn't yours, so neither tells you what users feel.

```bash
flutter run --profile
```

1. Open DevTools, then the **Performance** tab, and record about 5 seconds of the slow interaction.
2. Click a frame over budget. Note whether the build phase or the raster phase overran.
3. Turn on rebuild tracking and look for widgets that build far more often than the data changes.
4. Open the **CPU Profiler** and find the expensive functions on the UI thread.
5. Open the **Memory** tab, take a snapshot before and after repeating the flow, and compare them.

Add `showPerformanceOverlay: true` to `MaterialApp` for a live bar graph while you scroll; the same switch is in DevTools.

## Rebuild Reduction with select, const and child

Most build-phase jank comes from rebuilding more than changed. In Riverpod, `select` subscribes to one field, so a widget showing a name ignores every other change to the user. Bloc's equivalent is `buildWhen`.

```dart
// lib/widgets/user_name_label.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

/// Yours: bind this to your auth layer.
final userProvider = Provider<({String displayName, int points})>(
  (ref) => throw UnimplementedError(),
);

class UserNameLabel extends ConsumerWidget {
  const UserNameLabel({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final name = ref.watch(userProvider.select((user) => user.displayName));
    return Text(name);
  }
}
```

A `const` widget is one canonical instance, so Flutter stops at it during a rebuild of its parent. Put `const` on every static subtree. `flutter_lints` flags the ones you missed with `prefer_const_constructors` once you enable that rule.

`AnimatedBuilder` takes a `child` built once. Pass the expensive subtree there, or it rebuilds on every tick:

```dart
import 'dart:math';

import 'package:material_ui/material_ui.dart';

class SpinningLogo extends StatelessWidget {
  const SpinningLogo({super.key, required this.turns, required this.logo});

  final Animation<double> turns;
  final Widget logo;

  @override
  Widget build(BuildContext context) => AnimatedBuilder(
        animation: turns,
        child: logo, // built once by the caller, reused on every tick
        builder: (context, child) => Transform.rotate(
          angle: turns.value * 2 * pi,
          child: child,
        ),
      );
}
```

`RepaintBoundary` gives a subtree its own layer, so a spinner doesn't repaint the page behind it. Wrap a widget that repaints independently and often. List rows don't need it: `ListView.builder` already adds one per child through `addRepaintBoundaries`.

## Efficient Lists with Lazy Builders

`ListView(children: ...)` builds every row up front. `ListView.builder` builds what's on screen. When rows share a height, `itemExtent` lets the viewport compute positions without measuring each row. Mixed heights inside a scroll view with app bars go in `SliverList.builder`.

To keep a tab's scroll position and state across switches, mix `AutomaticKeepAliveClientMixin` into the tab's `State`, return `true` from `wantKeepAlive`, and call `super.build(context)` first in `build`. Do it only for tabs people return to: every kept tab holds its memory.

## Worked Example: A Feed That Scrolls at Full Rate

This screen parses off the UI thread, shows loading, error and empty states, decodes thumbnails at display size, and keeps rows cheap.

```dart
import 'dart:convert';

import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

class Article {
  const Article({required this.id, required this.title, required this.imageUrl});

  factory Article.fromJson(Map<String, dynamic> json) => Article(
        id: json['id'] as String,
        title: json['title'] as String,
        imageUrl: json['imageUrl'] as String,
      );

  final String id;
  final String title;
  final String imageUrl;
}

// Top-level, so compute can send it to another isolate.
List<Article> parseArticles(String body) => (jsonDecode(body) as List<dynamic>)
    .cast<Map<String, dynamic>>()
    .map(Article.fromJson)
    .toList(growable: false);

abstract interface class FeedApi {
  Future<String> fetchLatestJson();
}

/// Yours: bind this to your API client.
final feedApiProvider = Provider<FeedApi>((ref) => throw UnimplementedError());

final feedProvider = FutureProvider.autoDispose<List<Article>>((ref) async {
  final body = await ref.watch(feedApiProvider).fetchLatestJson();
  return compute(parseArticles, body);
});

class ArticleFeedScreen extends ConsumerWidget {
  const ArticleFeedScreen({super.key});

  static const double rowHeight = 88;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ref.watch(feedProvider).when(
          loading: () => const Center(child: CircularProgressIndicator()),
          error: (error, _) => Center(
            child: TextButton(
              onPressed: () => ref.invalidate(feedProvider),
              child: const Text('Could not load the feed. Try again'),
            ),
          ),
          data: (articles) => articles.isEmpty
              ? const Center(child: Text('Nothing here yet'))
              : ListView.builder(
                  itemCount: articles.length,
                  itemExtent: rowHeight,
                  itemBuilder: (context, index) =>
                      ArticleRow(article: articles[index]),
                ),
        );
  }
}

class ArticleRow extends StatelessWidget {
  const ArticleRow({super.key, required this.article});

  final Article article;
  static const double thumbnail = 72;

  @override
  Widget build(BuildContext context) {
    final decodeWidth =
        (thumbnail * MediaQuery.devicePixelRatioOf(context)).round();
    return ListTile(
      leading: CachedNetworkImage(
        imageUrl: article.imageUrl,
        width: thumbnail,
        height: thumbnail,
        fit: BoxFit.cover,
        memCacheWidth: decodeWidth,
        errorWidget: (context, url, error) =>
            const Icon(Icons.image_not_supported_outlined),
      ),
      title: Text(article.title, maxLines: 2, overflow: TextOverflow.ellipsis),
    );
  }
}
```

## Background Isolates for CPU Work

Move any work that can cross a frame budget off the UI isolate: JSON over about 100 KB, image processing, encryption, inference on custom models.

| Need | Use | Why |
|---|---|---|
| One job, mobile or desktop | `Isolate.run` | Short, typed, returns the result or rethrows |
| One job, and the app also runs on web | `compute` | On web it runs on the same thread rather than failing; `dart:isolate` isn't available there |
| Many jobs in a row | A long-lived isolate | Spawning has a cost, and a worker keeps its caches |
| A plugin call inside the isolate | Pass `RootIsolateToken.instance` and call `BackgroundIsolateBinaryMessenger.ensureInitialized` | Platform channels need a binary messenger in that isolate |

The closure given to `Isolate.run` copies whatever it captures. Capture plain data, never `this` of a widget or a state object, or you get `Illegal argument in isolate message`. A long-lived worker is in [references/isolate-worker.md](references/isolate-worker.md).

## Image Memory Limits

An image costs width × height × 4 bytes once decoded, whatever its file size. Decode at display size with `memCacheWidth` on `CachedNetworkImage`, or `ResizeImage` around any other provider. `maxWidthDiskCache` and `maxHeightDiskCache` shrink what the disk cache keeps, usually to twice the display size.

```dart
import 'package:material_ui/material_ui.dart';

Widget thumbnailOf(String url) => Image(
      image: ResizeImage(NetworkImage(url), width: 150, height: 150),
    );

void capImageCache() {
  PaintingBinding.instance.imageCache
    ..maximumSizeBytes = 50 * 1024 * 1024
    ..maximumSize = 200;
}
```

## App Size Reduction

`--split-per-abi` belongs to `flutter build apk` only. `flutter build appbundle` has no such flag and doesn't need one, because Google Play generates per-device APKs from the bundle. Use the APK flag when you distribute APKs yourself.

```bash
flutter build appbundle --release
flutter build apk --release --split-per-abi
flutter build apk --analyze-size --target-platform android-arm64
flutter build appbundle --analyze-size --target-platform android-arm64
```

`--analyze-size` works on release builds, on one ABI at a time, and can't be combined with `--split-debug-info`. Open the JSON it writes in DevTools' app size tool, and load two files to compare builds. The numbers aren't the store download. For that, upload the bundle and read Android vitals, or use Xcode's App Thinning Size Report on iOS.

| Tactic | Rough saving, measure yours |
|---|---|
| `--split-debug-info` on release builds | Can cut code size a lot |
| Remove unused packages | Varies; audit `pubspec.yaml` |
| Compress PNGs | 60 to 80% per image |
| WebP in place of PNG | 25 to 35% per image |
| Deferred components for rare features | 100 KB to 5 MB |
| Drop unused fonts | 1 to 3 MB |

### Deferred loading of heavy features

A deferred import loads a library on demand: `import 'package:my_app/features/ai/ai_chat.dart' deferred as ai_chat;`, then `await ai_chat.loadLibrary()` before the first use. On web that produces a separate `.js` file. On Android, the code only leaves the base module once you list the library under `flutter: deferred-components:` in `pubspec.yaml`, add the Play Core dependency and use `FlutterPlayStoreSplitApplication`; `flutter build appbundle` validates the setup and writes the missing files to `build/android_deferred_components_setup_files`. The official guide covers only Android and web, so don't count on a size win on iOS or desktop. Debug mode treats deferred imports as normal ones.

## Production Monitoring

DevTools sees one device. Firebase Performance traces, an HTTP interceptor, a startup trace, a memory-pressure handler and a rate-limited frame monitor are in [references/production-monitoring.md](references/production-monitoring.md), with the cold start checklist.

## Verify

The examples were compiled, not run on a device, so take every number from a profile build on real hardware.

```bash
flutter analyze
flutter test
flutter run --profile
```

Record the slow flow again and compare it with the numbers you took before the change. Then pin it with a scripted run on a real device, which writes its frame timings to `build/integration_response_data.json`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:material_ui/material_ui.dart';
import 'package:my_app/main.dart' as app;

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('feed scrolls', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    await binding.watchPerformance(() async {
      await tester.fling(find.byType(Scrollable), const Offset(0, -600), 3000);
      await tester.pumpAndSettle();
    }, reportKey: 'feed_scroll');
  });
}
```

```bash
flutter drive --profile --driver=test_driver/perf_driver.dart --target=integration_test/feed_scroll_test.dart
```

The driver file holds `import 'package:integration_test/integration_test_driver.dart'; Future<void> main() => integrationDriver();`.

## Gotchas

1. **Smooth in debug on the simulator, janky on a phone.** Symptom: nobody on the team sees the jank. Cause: the simulator's GPU and CPU are the Mac's. Fix: profile mode on the slowest device you support.
2. **`Could not find an option named "--split-per-abi"`.** Symptom: an appbundle build fails. Cause: the flag belongs to `flutter build apk`. Fix: drop it; Play splits a bundle by itself.
3. **`Cannot perform code size analysis when building for multiple ABIs`.** Symptom: `--analyze-size` stops at once. Cause: no single target. Fix: add `--target-platform android-arm64`.
4. **`Illegal argument in isolate message`.** Symptom: `Isolate.run` throws. Cause: the closure captured a `BuildContext`, a socket or `this`. Fix: copy plain values into local variables and capture those.
5. **`Bad state: The BackgroundIsolateBinaryMessenger.instance value is invalid`.** Symptom: a plugin call throws inside an isolate. Cause: the messenger was never set up. Fix: pass `RootIsolateToken.instance!` in and call `BackgroundIsolateBinaryMessenger.ensureInitialized(token)` first.
6. **List scrolls fine, then memory jumps.** Symptom: an out-of-memory kill in an image grid. Cause: 4000 px photos decoded for 100 px cells. Fix: `memCacheWidth` from `MediaQuery.devicePixelRatioOf`.
7. **Deferred import saved nothing on Android.** Symptom: the same AAB size. Cause: no `deferred-components` entry, so the code stays in the base module. Fix: complete the Play Feature Delivery setup.

## Anti-patterns

| Don't | Do instead |
|---|---|
| Optimize before recording a trace | Find the slow phase in DevTools first |
| Wrap every widget in `RepaintBoundary` | Wrap only what repaints often; each one costs a layer |
| `ListView(children: list.map(...).toList())` for long lists | `ListView.builder` |
| Parse big JSON in a provider on the UI isolate | `compute` or `Isolate.run` |
| Raise `imageCache` limits to stop reloads | Decode smaller, then keep the defaults or lower them |
| `--split-per-abi` on `appbundle` | `flutter build appbundle` as is |

## Pre-merge Checklist

- [ ] Before and after numbers from profile mode on a real device
- [ ] No frame over budget in the changed flow on the slowest supported device
- [ ] No work over one frame on the UI isolate
- [ ] Images in the changed screens decode at display size
- [ ] Long lists use a builder constructor
- [ ] Memory returns to its baseline after repeating the flow
- [ ] Download size checked against the 20 MB target when dependencies or assets changed

## See Also

- `flutter-animations` for motion that stays inside the budget
- `flutter-state` for provider design and `select`
- `flutter-fix-layout-issues` for overflow and layout errors
- `flutter-analytics-firebase` for the Firebase setup behind monitoring
