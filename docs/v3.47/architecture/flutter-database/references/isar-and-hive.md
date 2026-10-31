---
title: Isar Community and Hive CE
author: MOZGII, Inc.
lang: en
tags: [flutter, isar, hive, nosql, preferences]
description: Fast indexed object storage with Isar Community, and key-value preferences with Hive CE, both on their maintained forks.
---
# Isar Community and Hive CE

Both stores here are maintained forks. Old guides still point at the originals, which no longer work with current Dart.

## Isar Community: NoSQL Embedded Database

Isar is best for non-relational object storage where query speed is critical. The original `isar` package has had no release since April 2023 and declares an upper Dart bound below 3, so use the maintained fork `isar_community`, with `isar_community_flutter_libs` and `import 'package:isar_community/isar.dart';`. Schemas are generated, so `isar_community_generator` goes in dev dependencies.

```yaml
dependencies:
  isar_community: ^3.4.1
  isar_community_flutter_libs: ^3.4.1
  path_provider: ^2.1.6
dev_dependencies:
  isar_community_generator: ^3.4.1
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
```

## Isar Indexes and Where Clauses

`where()` only offers indexed properties, and `filter()` works on any property by scanning. So `isFavorite` carries `@Index()`, and without it `.where().isFavoriteEqualTo(true)` doesn't exist. A composite index on `authorId` and `createdAt` serves "this author's items, newest first" from the index with no sort step, and its generated clause is `authorIdEqualToAnyCreatedAt`. Leave `authorId` on the default hashed index type: only the last property of a composite index may be a non-hashed `String`, and `type: IndexType.value` here stops the generator.

`@Index(unique: true, replace: true)` on `remoteId` is what makes `put` an upsert: a new object with an existing `remoteId` replaces the row, even though its auto-incremented `id` differs. Without it, every sync adds a duplicate.

```dart
// lib/features/media/data/models/media_item_isar.dart
import 'package:isar_community/isar.dart';

part 'media_item_isar.g.dart';

@collection
class MediaItemIsar {
  Id id = Isar.autoIncrement;

  @Index(unique: true, replace: true)
  late String remoteId;

  late String title;
  late String thumbnailUrl;

  @Index(composite: [CompositeIndex('createdAt')])
  late String authorId;

  late DateTime createdAt;

  @Index()
  late bool isFavorite;
}
```

```dart
// lib/features/media/data/datasources/media_local_datasource.dart
import 'package:isar_community/isar.dart';
import 'package:path_provider/path_provider.dart';

import '../models/media_item_isar.dart';

Future<Isar> openIsar() async {
  final dir = await getApplicationDocumentsDirectory();
  return Isar.open(
    [MediaItemIsarSchema],
    directory: dir.path,
    name: 'app_isar',
  );
}

class MediaLocalDataSource {
  MediaLocalDataSource(this._isar);
  final Isar _isar;

  Future<List<MediaItemIsar>> getByAuthor(String authorId) => _isar
      .mediaItemIsars
      .where(sort: Sort.desc)
      .authorIdEqualToAnyCreatedAt(authorId)
      .findAll();

  Stream<List<MediaItemIsar>> watchFavorites() => _isar.mediaItemIsars
      .where()
      .isFavoriteEqualTo(true)
      .watch(fireImmediately: true);

  Future<void> upsert(MediaItemIsar item) =>
      _isar.writeTxn(() => _isar.mediaItemIsars.put(item));

  Future<void> upsertAll(List<MediaItemIsar> items) =>
      _isar.writeTxn(() => _isar.mediaItemIsars.putAll(items));
}
```

## Hive CE: Key-Value Preferences

Hive is for simple structured preferences, not relational data or large datasets. Use Hive CE: the `hive` 2.x line stopped at 2.2.3 in 2022, and the `hive: ^4.0.2` pin older guides show was never published as a stable release. The packages are `hive_ce`, `hive_ce_flutter` and `hive_ce_generator`; import `package:hive_ce_flutter/hive_ce_flutter.dart`.

```yaml
dependencies:
  hive_ce_flutter: ^2.5.0
```

Initialise it in `main()` with `await Hive.initFlutter();` and `await Hive.openBox('settings');`. After that, `Hive.box('settings')` is synchronous, which is what lets a preference apply before the first frame. Use it only for booleans, ints, strings and simple value types. For complex objects, use Drift or Isar.

```dart
// lib/core/storage/settings_storage.dart
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'package:material_ui/material_ui.dart';

class SettingsStorage {
  final Box<dynamic> _box = Hive.box('settings');

  static const _themeKey = 'theme_mode';
  static const _languageKey = 'language';
  static const _onboardedKey = 'has_seen_onboarding';

  ThemeMode get themeMode =>
      ThemeMode.values[_box.get(_themeKey, defaultValue: 0) as int];

  set themeMode(ThemeMode mode) => _box.put(_themeKey, mode.index);

  String? get language => _box.get(_languageKey) as String?;

  set language(String? value) => _box.put(_languageKey, value);

  bool get hasSeenOnboarding =>
      _box.get(_onboardedKey, defaultValue: false) as bool;

  set hasSeenOnboarding(bool value) => _box.put(_onboardedKey, value);
}
```
