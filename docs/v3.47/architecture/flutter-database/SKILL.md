---
name: flutter-database
description: Builds the persistence and sync layer of an offline-first Flutter app - choosing Drift, Isar Community, Hive CE, Supabase Realtime or Firestore, a repository that reads local first, a durable sync queue, conflict policies, and gRPC streaming into Drift. Use when adding local storage, an offline mode, real-time subscriptions, a sync queue, conflict handling between devices, or user preference storage.
metadata:
  tags:
  - flutter
  - database
  - drift
  - isar
  - supabase
  - firestore
  - offline-first
---
# Flutter Database

An offline-first app always reads from local storage and syncs in the background, so it works with zero network connectivity. The repository interface is the only contract between business logic and storage, there are no raw SQL strings in application code, and every write to remote is queued and survives an app restart. Done this way, a tunnel is a non-event rather than a spinner. Use it for any local persistence, real-time subscriptions, offline-first sync, a repository bridging local and remote data, or preference storage. Skip it when you only read from an API with no local caching, where a Riverpod `AsyncNotifier` and Dio are enough, and for auth session storage, which is `flutter-auth`'s job.

## Step 0: Read Your Versions

```bash
flutter --version
flutter pub deps --style=compact | grep -E "^- (drift|sqlite3|isar_community|hive_ce|supabase_flutter|cloud_firestore) "
grep -n "sqlite3_flutter_libs\|sqlcipher_flutter_libs" pubspec.yaml       # both are end of life
grep -n "schemaVersion" -r lib                                             # note it before changing a table
```

The pins were read from pub.dev on 2026-10-30, and the examples were compiled against them with Flutter 3.47.9 and Dart 3.13.7. The gRPC examples use real `protoc` output. The tests ran on the host and the app flows were compiled, not run on a device, so do the airplane-mode check in Verify on a real phone. `references/isar-and-hive.md` imports `material_ui`: on `package:flutter/material.dart`, swap that import, or run `dart fix --apply --code=migrate_design_widgets`.

```yaml
dependencies:
  drift: ^2.35.3
  drift_flutter: ^0.3.1
  connectivity_plus: ^7.3.1    # below 7.3.2: its dbus 0.8 conflicts with geolocator
  fpdart: ^1.2.0
  freezed_annotation: ^3.1.0
  injectable: ^3.0.0
  path_provider: ^2.1.6
  path: ^1.9.1
  riverpod_annotation: ^4.0.7

dev_dependencies:
  drift_dev: ^2.35.2
  build_runner: ^2.16.1    # below 2.17.1: most generators still need build 4
  freezed: ^4.0.3
  injectable_generator: ^3.1.3    # writes injection.config.dart
  riverpod_generator: ^4.0.9    # writes the .g.dart files of @riverpod providers
```

Since drift 2.32, the `sqlite3` package is on its 3.x line, which builds SQLite through build hooks. `sqlite3_flutter_libs` (`0.6.0+eol`) and `sqlcipher_flutter_libs` (`0.7.0+eol`) are empty end-of-life releases, so remove them from your pubspec. Encryption is in `flutter-security`. The examples also import `material_ui: ^1.9.0`.

## Choosing a Store

| Need | Pick | Why the others lose |
|---|---|---|
| Relational data, joins, migrations | **Drift** (type-safe SQLite) | Isar has no joins; Hive has no queries |
| Fast object storage with indexed queries, no relations | **Isar Community** | Drift needs schema and SQL thinking; the original `isar` is unmaintained |
| Small preferences read at startup | **Hive CE** | A full database is too much for a theme flag |
| Live data shared between users on Postgres | **Supabase Realtime** | Firestore needs a document model |
| Cloud documents with built-in offline cache | **Firestore** | Supabase has no client offline cache |
| High-throughput live feeds | **gRPC streaming into Drift** | Realtime channels aren't built for it |

Setup for Isar and Hive is in `references/isar-and-hive.md`, and Supabase and Firestore are in `references/supabase-and-firestore.md`. For encrypted secrets use secure storage, never these.

## Worked Example: An Offline-First Repository on Drift

Drift is the primary local relational database for Flutter: type-safe, code-generated, zero raw SQL. These files give a screen data that keeps working offline, and `references/repository-pattern.md` holds the `Failure` type they report with.

### Drift tables, DAOs and migrations

Tables are Dart classes, and DAOs hold the queries. `schemaVersion` starts at 1 and goes up by one with each change; `onUpgrade` adds what's new, and `onCreate` builds everything for a fresh install. Bumping the version without a migration, or the reverse, breaks existing users. `driftDatabase()` from `drift_flutter` opens a file in the app's documents folder on a background isolate. To open it yourself, use `NativeDatabase.createInBackground(file)` inside a `LazyDatabase`, with `path_provider` and `path`. The constructor takes a `QueryExecutor` so tests can pass an in-memory database. `textEnum<SyncStatus>()` stores the enum's `.name`, and the generated Dart field is the enum itself. So a companion takes `Value(SyncStatus.synced)` and never a `Value('synced')`.

```dart
// lib/core/database/app_database.dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

import 'messages_dao.dart';

part 'app_database.g.dart';

enum SyncStatus {
  synced, // local == remote
  localOnly, // created offline, not yet pushed
  modified, // local changes pending push
  conflicted, // local and remote both changed since last sync
}

class PostsTable extends Table {
  TextColumn get id => text()();
  TextColumn get title => text().withLength(min: 1, max: 500)();
  TextColumn get body => text()();
  TextColumn get authorId => text()();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  BoolColumn get isSynced => boolean().withDefault(const Constant(false))();
  TextColumn get syncStatus =>
      textEnum<SyncStatus>().withDefault(Constant(SyncStatus.localOnly.name))();
  DateTimeColumn get updatedAt => dateTime().withDefault(currentDateAndTime)();
  DateTimeColumn get serverUpdatedAt => dateTime().nullable()();

  @override
  Set<Column<Object>> get primaryKey => {id};
}

class SyncOperationsTable extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get operation => text()();
  TextColumn get payload => text()();
  TextColumn get status => text().withDefault(const Constant('pending'))();
  IntColumn get attempts => integer().withDefault(const Constant(0))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}

class AccountsTable extends Table {
  TextColumn get id => text()();
  TextColumn get displayName => text()();
  TextColumn get subscriptionTier => text()();
  IntColumn get accountBalance => integer()();
  TextColumn get syncStatus =>
      textEnum<SyncStatus>().withDefault(Constant(SyncStatus.localOnly.name))();

  @override
  Set<Column<Object>> get primaryKey => {id};
}

class MessagesTable extends Table {
  TextColumn get id => text()();
  TextColumn get roomId => text()();
  TextColumn get content => text()();
  IntColumn get timestampMs => integer()();

  @override
  Set<Column<Object>> get primaryKey => {id};
}

@DriftAccessor(tables: [PostsTable])
class PostsDao extends DatabaseAccessor<AppDatabase> with _$PostsDaoMixin {
  PostsDao(super.db);

  Future<List<PostsTableData>> getAllPosts() => select(postsTable).get();

  Stream<List<PostsTableData>> watchAllPosts() => (select(
    postsTable,
  )..orderBy([(t) => OrderingTerm.desc(t.createdAt)])).watch();

  Stream<List<PostsTableData>> watchConflicted() => (select(
    postsTable,
  )..where((t) => t.syncStatus.equalsValue(SyncStatus.conflicted))).watch();

  Future<PostsTableData?> getById(String id) =>
      (select(postsTable)..where((t) => t.id.equals(id))).getSingleOrNull();

  /// Rows with local changes the server hasn't accepted yet.
  Future<List<String>> unsyncedIds() =>
      (select(postsTable)
            ..where((t) => t.syncStatus.equalsValue(SyncStatus.synced).not()))
          .map((row) => row.id)
          .get();

  Future<void> markSynced(String id) =>
      (update(postsTable)..where((t) => t.id.equals(id))).write(
        const PostsTableCompanion(syncStatus: Value(SyncStatus.synced)),
      );

  Future<void> upsertPost(Insertable<PostsTableData> post) =>
      into(postsTable).insertOnConflictUpdate(post);

  Future<void> upsertPosts(List<PostsTableCompanion> posts) =>
      batch((b) => b.insertAllOnConflictUpdate(postsTable, posts));

  Future<int> deletePost(String id) =>
      (delete(postsTable)..where((t) => t.id.equals(id))).go();
}

@DriftAccessor(tables: [SyncOperationsTable])
class SyncQueueDao extends DatabaseAccessor<AppDatabase>
    with _$SyncQueueDaoMixin {
  SyncQueueDao(super.db);

  Future<int> enqueue(SyncOperationsTableCompanion op) =>
      into(syncOperationsTable).insert(op);

  Future<List<SyncOperationsTableData>> getPending() =>
      (select(syncOperationsTable)
            ..where((t) => t.status.equals('pending'))
            ..orderBy([(t) => OrderingTerm.asc(t.id)]))
          .get();

  Future<void> markComplete(int id) =>
      (update(syncOperationsTable)..where((t) => t.id.equals(id))).write(
        const SyncOperationsTableCompanion(status: Value('completed')),
      );

  Future<void> recordFailure(int id, int attempts) =>
      (update(syncOperationsTable)..where((t) => t.id.equals(id))).write(
        SyncOperationsTableCompanion(attempts: Value(attempts)),
      );

  /// Takes an operation the server will never accept out of the drain.
  Future<void> park(int id, int attempts) =>
      (update(syncOperationsTable)..where((t) => t.id.equals(id))).write(
        SyncOperationsTableCompanion(
          status: const Value('parked'),
          attempts: Value(attempts),
        ),
      );

  /// What the person has to see: these writes will not be sent again.
  Stream<List<SyncOperationsTableData>> watchParked() =>
      (select(syncOperationsTable)..where((t) => t.status.equals('parked')))
          .watch();

  Future<List<String>> pendingPayloads(String operation) =>
      (select(syncOperationsTable)..where(
            (t) => t.status.equals('pending') & t.operation.equals(operation),
          ))
          .map((op) => op.payload)
          .get();

  /// The newest operation the server has confirmed, or 0.
  Future<int> lastCompletedId() {
    final newest = syncOperationsTable.id.max();
    final query = selectOnly(syncOperationsTable)
      ..addColumns([newest])
      ..where(syncOperationsTable.status.equals('completed'));
    return query.map((row) => row.read(newest) ?? 0).getSingle();
  }

  /// Payloads of [operation] queued after the operation [id], sent or not.
  Future<List<String>> payloadsAfter(String operation, int id) =>
      (select(syncOperationsTable)..where(
            (t) => t.id.isBiggerThanValue(id) & t.operation.equals(operation),
          ))
          .map((op) => op.payload)
          .get();
}

@DriftDatabase(
  tables: [PostsTable, SyncOperationsTable, AccountsTable, MessagesTable],
  daos: [PostsDao, SyncQueueDao, MessagesDao],
)
class AppDatabase extends _$AppDatabase {
  AppDatabase([QueryExecutor? executor])
    : super(executor ?? driftDatabase(name: 'app'));

  @override
  int get schemaVersion => 2;

  @override
  MigrationStrategy get migration => MigrationStrategy(
    onCreate: (migrator) => migrator.createAll(),
    onUpgrade: (migrator, from, to) async {
      if (from < 2) {
        // addColumn would fail: updatedAt's default is not a constant.
        await migrator.alterTable(
          TableMigration(
            postsTable,
            columnTransformer: {
              // Rows the server already has must not turn localOnly.
              postsTable.syncStatus: const CustomExpression<String>(
                "CASE is_synced WHEN 1 THEN 'synced' ELSE 'localOnly' END",
              ),
            },
            newColumns: [
              postsTable.syncStatus,
              postsTable.updatedAt,
              postsTable.serverUpdatedAt,
            ],
          ),
        );
        await migrator.createTable(syncOperationsTable);
        await migrator.createTable(accountsTable);
        await migrator.createTable(messagesTable);
        // Unsent rows need a queue entry, or nothing ever pushes them.
        await customStatement(
          "INSERT INTO sync_operations_table (operation, payload) "
          "SELECT 'post.create', id FROM posts_table WHERE is_synced = 0",
        );
      }
    },
  );
}
```

Run `dart run build_runner build` after schema changes. `alterTable` with a `TableMigration` rebuilds the table and keeps its rows, which is the way to add a column whose default is an expression such as `currentDateAndTime`. SQLite's `ALTER TABLE ADD COLUMN` only takes constant defaults, so `addColumn` works for `syncStatus` and `serverUpdatedAt` but not for `updatedAt`. The `columnTransformer` carries `is_synced` over into `syncStatus`, because the background sync skips every row that isn't `synced`: left to the column default, each upgraded row would turn `localOnly` and never refresh. Rows that were never sent get a `post.create` queue entry in the same step. `references/database-tests.md` runs this upgrade on a version 1 schema.

### The repository pattern: local first, sync behind

Define the repository interface in the domain layer before implementing storage: `getPosts`, `getPostById`, `createPost`, `deletePost` and a live `watchPosts`, each returning `Either<Failure, T>` or a stream. The implementation returns local data at once and syncs from remote in the background. A failed sync is silent because the local data has already been returned, and the UI gets the update through `watchPosts`, a Drift stream. A write changes local storage and enqueues the remote call in one Drift transaction, so a crash can't leave one without the other, then starts a drain, so an online write goes out at once. The background sync skips rows with unsent changes and rows with a queued delete, including a delete that finished while the fetch was running, so it can't undo the person's work. The full class is in `references/repository-pattern.md`.

## Offline Sync Queue That Survives Restarts

Writes that must survive network outages go through a queue stored in Drift and retried on network restore. `onConnectivityChanged` emits a `List<ConnectivityResult>`, so compare with `contains`: `result != ConnectivityResult.none` on a list is always true, and the drain would run with no network. `SyncService` drains in order, one drain at a time, and stops at the first failure, because the next write will fail too and order matters. A write made during a drain can't start a second one, so the running drain reads the queue again until it's empty. The exception is a write the server will never accept. `_send` reports it by throwing `PermanentSyncFailure`, which parks the operation so it can't block the queue forever; show parked writes to the person through `watchParked`, since they may have to redo them. `start()` also drains once, because a connectivity event only fires on a change: an app that opens online with writes left from its last run would otherwise never send them. Each operation may be delivered twice if the app dies after the server replied, so your server has to treat them as idempotent.

`SyncService` is in `references/sync-queue.md`.

## Sync Conflict Resolution

When the same record is modified locally and remotely, a conflict policy must exist. Choose one policy per entity type and don't mix them. The `syncStatus`, `updatedAt` and `serverUpdatedAt` columns on `PostsTable` are what every table that syncs needs, and each policy's code is in `references/conflict-policies.md`.

| Policy | Use when | Example |
|---|---|---|
| Last write wins | Data loss is acceptable | Profile fields, settings |
| Server wins | The server owns the field | Balance, subscription status, permissions |
| Conflict flag + user resolution | Data loss is unacceptable | Notes, documents, forms |

## Verify

```bash
dart run build_runner build
flutter analyze
flutter test
grep -rn "result != ConnectivityResult" lib             # expect nothing: the event is a List
grep -rnE "customSelect|customStatement" lib             # raw SQL belongs in migrations only
```

`references/database-tests.md` has tests that need no device: they run the schema on in-memory SQLite, and prove the version 1 to 2 upgrade, the enum storage, queue order, the stop on failure, parking, a write sent right after its transaction, a write made during a drain, a sync that leaves local work alone, a reconnect loop that ends, and a cancel that returns at once. Then check on a device: turn on airplane mode, delete a post, kill the app, reopen it and turn the network back on. The delete must reach the server exactly once. Export a schema with `dart run drift_dev make-migrations` before shipping a schema change, and test the upgrade from each released version.

## Gotchas

1. **The queue drains with no network.** Symptom: every queued write fails while offline. Cause: `result != ConnectivityResult.none` on a `List` is always true. Fix: `results.contains(ConnectivityResult.none)`.
2. **`The argument type 'String' can't be assigned to the parameter type 'SyncStatus'`.** Symptom: `Value('synced')` fails on a `textEnum` column. Cause: the generated field is the enum. Fix: `Value(SyncStatus.synced)`.
3. **`no such column: sync_status` for existing users.** Symptom: fresh installs work and upgrades crash. Cause: a new column with no `onUpgrade` step. Fix: bump `schemaVersion` and add the step.
4. **`Cannot add a column with non-constant default`.** Symptom: an upgrade crashes on `addColumn`, while fresh installs and tests that start from `onCreate` pass. Cause: SQLite's `ALTER TABLE ADD COLUMN` refuses a default like `currentDateAndTime`. Fix: `migrator.alterTable(TableMigration(table, newColumns: [...]))`, and test the upgrade from the old schema.
5. **Old rows throw after renaming an enum value.** Symptom: `ArgumentError` reading a row. Cause: `textEnum` stores the name. Fix: migrate the stored strings, or add new values instead of renaming.
6. **`The method 'isFavoriteEqualTo' isn't defined for the type 'QueryBuilder'`.** Symptom: an Isar `where()` won't compile. Cause: `where()` only offers indexed properties. Fix: add `@Index()`, or use `.filter()`. The related `Only the last property of a composite index may be a non-hashed String` means removing `type: IndexType.value` from a composite index on a `String`.
7. **`Target of URI doesn't exist: 'chat.v1.pb.dart'`.** Symptom: an import of generated gRPC code fails. Cause: `protoc` names files after the `.proto` file, not its `package`. Fix: import `chat.pbgrpc.dart`.
8. **`The argument type 'int' can't be assigned to the parameter type 'Int64?'`.** Symptom: a proto3 `int64` field won't take an `int`. Cause: protobuf uses `Int64` from `fixnum`. Fix: `Int64(value)`, and `.toInt()` to read it.
9. **A gRPC stream reconnects forever.** Symptom: a tight loop against the server after a token expires. Cause: the retry re-sent the same token. Fix: force a refresh once, then surface the error.
10. **`reconnect()` never returns, or `Bad state: Cannot add event after closing` after `watchRoom`.** Symptom: `cancel()` hangs in a silent room or with the server down, or an exception follows leaving a chat. Cause: an `async*` loop only sees a cancel at its next `yield`, and a retry timer can outlive its controller. Fix: stop the call and the timer in the `StreamController`'s `onCancel`, as `references/grpc-streaming.md` does.

## Anti-Patterns

| Don't | Do instead |
|---|---|
| Read from the network first | Read local, sync behind |
| Write straight to the server from a screen | Write local, enqueue, drain |
| Leave a queued write for the next connectivity event | Start a drain after each transaction: the event only fires on a change |
| Parse raw SQL in application code | Drift's typed queries |
| Trust client clocks for last write wins on anything valuable | Server timestamps, or the flag policy |
| Put large objects in Hive | Drift or Isar |
| Store secrets in any of these | `flutter_secure_storage` |
| Close a shared gRPC channel from a provider's `onDispose` | Cancel the stream; the channel is a singleton |

## Pre-Merge Checklist

- [ ] The app works in airplane mode, including a cold start
- [ ] Every remote write goes through the queue, in one transaction with the local change, and survives a restart
- [ ] The queue drains at start and after each write, and a permanent failure is parked and shown to the person
- [ ] `schemaVersion` went up with each table change, and the upgrade path is tested
- [ ] Each syncing entity has one named conflict policy
- [ ] Connectivity events are read as a `List`
- [ ] Streams and channels are cancelled when their owner is disposed, and reconnect loops have backoff and a way to end
- [ ] Repositories return entities and `Failure`, not rows or protos

## See Also

`flutter-network` (API clients, an offline queue over any store), `flutter-state` (Hive-backed startup state), `flutter-security` (database encryption, secure storage), `flutter-core` (architecture), `flutter-auth` (session storage). Depth in this skill: `references/repository-pattern.md`, `references/sync-queue.md`, `references/isar-and-hive.md`, `references/supabase-and-firestore.md`, `references/conflict-policies.md`, `references/grpc-streaming.md`, `references/database-tests.md`.
