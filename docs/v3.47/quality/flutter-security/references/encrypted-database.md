---
title: "Encrypting a Flutter Drift database with SQLCipher"
author: MOZGII, Inc.
lang: en
tags: [flutter, security, sqlite, sqlcipher, drift, encryption]
description: Opens a Drift database through the sqlite3 3.x SQLCipher build - the hook define, a random key in secure storage, a runtime check that encryption is really on, and a one-time migration from a plain file.
---
# Encrypted Database with SQLCipher

`sqlite3` 3.x loads SQLite through a build hook, and a user define in `pubspec.yaml` selects the encrypted build. `sqlcipher_flutter_libs` and `sqlite3_flutter_libs` reached end of life with 3.x: their last releases hold no code, so remove them.

```yaml
dependencies:
  drift: ^2.35.3
  sqlite3: ^3.8.0
  flutter_secure_storage: ^11.2.0

hooks:
  user_defines:
    sqlite3:
      source: sqlcipher   # or sqlite3mc for SQLite3MultipleCiphers
```

`source: sqlcipher` may bundle an older SQLite than the default build, and it links OpenSSL on Android, Windows and Linux. Both encrypted builds carry their own licences. `sqlite3mc` needs extra pragmas to open a database SQLCipher wrote.

## Generate the Database Key

The key is 32 random bytes from `Random.secure`, stored once in secure storage and never derived from anything on the device.

```dart
// lib/core/database/database_key.dart
import 'dart:convert';
import 'dart:math';

import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const _keyName = 'db_encryption_key';

Future<String> getOrCreateDatabaseKey(FlutterSecureStorage storage) async {
  final existing = await storage.read(key: _keyName);
  if (existing != null) return existing;

  final random = Random.secure();
  final bytes = List<int>.generate(32, (_) => random.nextInt(256));
  final key = base64Url.encode(bytes);
  await storage.write(key: _keyName, value: key);
  return key;
}
```

## Open the Encrypted Database and Prove It

The check has to be a real one. `assert` is stripped from release builds, and plain SQLite treats `PRAGMA key` as a no-op, so a missing hook define would write an unencrypted file with no error at all.

```dart
// lib/core/database/app_database.dart
import 'dart:io';

import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:sqlite3/sqlite3.dart';

/// Throws in every build mode when the encrypted SQLite isn't loaded.
void requireSqlCipher(Database database) {
  final version = database.select('PRAGMA cipher_version;');
  if (version.isEmpty || version.first.columnAt(0) == null) {
    throw StateError(
      'SQLCipher is not loaded: check hooks.user_defines.sqlite3.source',
    );
  }
}

QueryExecutor openEncryptedDatabase(String encryptionKey) {
  return LazyDatabase(() async {
    final dir = await getApplicationDocumentsDirectory();
    final file = File(p.join(dir.path, 'app.db'));

    return NativeDatabase.createInBackground(
      file,
      setup: (database) {
        requireSqlCipher(database);
        // The key goes first, before anything else touches the file.
        final escaped = encryptionKey.replaceAll("'", "''");
        database.execute("PRAGMA key = '$escaped';");
        database.execute('PRAGMA cipher_page_size = 4096;');
        database.execute('PRAGMA kdf_iter = 256000;');
      },
    );
  });
}
```

The `setup` callback runs in the background isolate, so keep it free of anything that needs the main isolate. Pass the executor to your generated database class: `AppDatabase(openEncryptedDatabase(key))`.

## Migrate a Plain Database to Encrypted

Run this at each launch, before the database opens; it does nothing once the migration is done. Your app has been writing to `app.db`, so the first step moves that plain file to `app_plain.db`, with its journal files. The header tells the two apart: a plain SQLite file starts with `SQLite format 3`, and an encrypted one doesn't. The export then writes a new `app.db`, and only after that does the plain copy go, journal files first. `app_plain.db` still exists if the app dies partway, so the next launch discards the partial export and starts again.

```dart
// lib/core/database/migrate_to_encrypted.dart
import 'dart:convert';
import 'dart:io';

import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:sqlite3/sqlite3.dart';

const _journals = ['-wal', '-shm', '-journal'];

bool _isPlainSqlite(File file) {
  if (!file.existsSync() || file.lengthSync() < 16) return false;
  final header = file.openSync();
  try {
    return latin1.decode(header.readSync(15)) == 'SQLite format 3';
  } finally {
    header.closeSync();
  }
}

Future<void> migrateToEncrypted(String encryptionKey) async {
  final dir = await getApplicationDocumentsDirectory();
  final plainPath = p.join(dir.path, 'app_plain.db');
  final encryptedPath = p.join(dir.path, 'app.db');

  if (_isPlainSqlite(File(encryptedPath))) {
    for (final suffix in ['', ..._journals]) {
      final file = File('$encryptedPath$suffix');
      if (file.existsSync()) file.renameSync('$plainPath$suffix');
    }
  }
  if (!File(plainPath).existsSync()) return; // Already migrated.

  // A run that died partway left a partial export.
  for (final suffix in ['', ..._journals]) {
    final file = File('$encryptedPath$suffix');
    if (file.existsSync()) file.deleteSync();
  }

  final escapedKey = encryptionKey.replaceAll("'", "''");
  final escapedPath = encryptedPath.replaceAll("'", "''");
  final database = sqlite3.open(plainPath);
  try {
    database.execute(
      "ATTACH DATABASE '$escapedPath' AS encrypted KEY '$escapedKey';",
    );
    database.execute("SELECT sqlcipher_export('encrypted');");
    database.execute('DETACH DATABASE encrypted;');
  } finally {
    database.close();
  }

  for (final suffix in [..._journals, '']) {
    final file = File('$plainPath$suffix');
    if (file.existsSync()) file.deleteSync();
  }
}
```

## Verify the File Is Encrypted

Pull the file from a test device and look at its first bytes. Any unencrypted SQLite file starts with the text `SQLite format 3`.

```bash
adb exec-out run-as com.example.app cat app_flutter/app.db | head -c 16 | xxd
# expect random bytes, never "SQLite format 3"
```

## Gotchas for Encrypted Databases

1. **A file with plain text in it.** Symptom: `SQLite format 3` at the start of the file. Cause: the hook define is missing or sits in a dependency's pubspec. Fix: put `hooks.user_defines` in the app's own `pubspec.yaml`, and keep `requireSqlCipher`.
2. **`file is not a database`.** Symptom: an open fails on existing data. Cause: the key changed, or the secure store was reset. Fix: treat the local database as a cache you can rebuild from the server, or back up the key.
3. **Secure storage reset wipes the data.** Symptom: everything disappears after a Keystore problem. Cause: `resetOnError` removed the key. Fix: design for it, and never keep the only copy of user data here.
