---
name: dart-path-handling
description: Handles file paths safely on every platform with package:path and package:file - joining, splitting, extensions, normalizing, containment checks against path traversal, POSIX and URL conversion, and testing with an in-memory file system. Use when asked to fix a path that breaks on Windows, join paths, read a file extension, stop a path escaping a folder, or test code that touches the disk.
metadata:
  tags:
  - dart
  - paths
  - cross-platform
---
# Cross-Platform Paths in Dart

A string built with `/` works on your Mac and fails on a colleague's Windows machine. `package:path` knows each platform's separators and roots, and `package:file` lets tests run against an in-memory disk. Together they remove a whole class of bugs that only shows up on another OS.

## Step 0: Find the Fragile Code

```bash
dart --version
grep -rnE "'\\\$[a-zA-Z_.]+/|endsWith\('\.|\.split\('/'\)|lastIndexOf\('\.'\)|replaceAll\(r?'\\\\" lib bin tool 2>/dev/null
dart pub add path file
```

Each hit is worth reading: the pattern is usually one of the replacements below. Import with a prefix: `import 'package:path/path.dart' as p;`. The examples also import `test: ^1.33.0`.

## String Habits to Replace

| Goal | Write | Not | Why |
|---|---|---|---|
| Join | `p.join(dir, name)` | `'$dir/$name'` | Wrong separator on Windows, doubled slashes |
| Base name | `p.basename(path)` | `path.split('/').last` | Fails on backslashes |
| Extension | `p.extension(path) == '.png'` | `path.endsWith('.png')` | A folder can end in `.png` |
| Compound extension | `p.extension(path, 2)` | `lastIndexOf('.')` | Breaks on `.tar.gz` and dotfiles |
| Without extension | `p.withoutExtension(path)` | `substring` | Same reason |
| Inside a folder | `p.isWithin(root, path)` | `path.startsWith(root)` | `/data2` starts with `/data` |
| To a URI | `p.toUri(path)` | `Uri.parse(path)` | Fails on `C:` and spaces |
| From a URI | `p.fromUri(uri)` | `uri.path` | Leaves `%20` in the name |
| To URL form | `p.url.joinAll(p.split(path))` | `replaceAll(r'\', '/')` | Mishandles drive roots |
| Prefix segments | `if (p.split(path) case ['assets', ...final rest])` | `p.split(path).first` | `.first` throws on an empty list |

## Which Context

The top-level `p.*` functions use the current platform's rules. `p.posix`, `p.windows` and `p.url` are fixed contexts for formats that don't depend on the host, such as Git paths or URLs. Build a `p.Context(style: p.Style.windows)` to unit test Windows behaviour on a Mac.

| Data | Use | Why the others lose here |
|---|---|---|
| A file on this machine | `p.*` | A fixed context breaks on another OS |
| A path inside a URL | `p.url.*` | The host's separator would leak in |
| A path in Git or a manifest | `p.posix.*` | Those formats always use `/` |
| A test for Windows logic | `p.Context(style: p.Style.windows)` | You'd need a Windows machine otherwise |
| Code that reads and writes files in tests | `package:file` | Real disk tests are slow and leak state |

## Worked Example: Saving a Receipt Safely

A receipt's file name comes from outside the app, so a name such as `../../secrets.txt` must not escape the folder. Normalize first, then check containment. The `FileSystem` is injected, so the test needs no disk.

```dart
// lib/receipt_store.dart
import 'dart:typed_data';

import 'package:file/file.dart';
import 'package:path/path.dart' as p;

class UnsafePath implements Exception {
  const UnsafePath(this.name);
  final String name;
}

class ReceiptStore {
  ReceiptStore(this._fs, this._root);

  final FileSystem _fs;
  final String _root;

  /// Writes [bytes] under the store's root and returns the file's path.
  ///
  /// Throws [UnsafePath] when [name] would land outside the root.
  Future<String> save(String name, Uint8List bytes) async {
    final target = p.normalize(p.absolute(_root, name));
    if (!p.isWithin(p.normalize(p.absolute(_root)), target)) {
      throw UnsafePath(name);
    }
    final file = _fs.file(target);
    await file.parent.create(recursive: true);
    await file.writeAsBytes(bytes);
    return target;
  }

  /// Lists stored receipts of one [extension], such as `.pdf`.
  Future<List<String>> list(String extension) async => [
        await for (final entity in _fs.directory(_root).list(recursive: true))
          if (entity is File && p.extension(entity.path) == extension) entity.path,
      ];
}
```

```dart
// lib/main_store.dart: production wiring
import 'package:expense_tracker/receipt_store.dart';
import 'package:file/local.dart';

final store = ReceiptStore(const LocalFileSystem(), '/var/app/receipts');
```

`p.join(_root, name)` alone isn't safe: when `name` is absolute, `join` discards the root. Always verify the result.

## Testing Without the Disk

```dart
import 'dart:typed_data';

import 'package:expense_tracker/receipt_store.dart';
import 'package:file/memory.dart';
import 'package:path/path.dart' as p;
import 'package:test/test.dart';

void main() {
  late ReceiptStore store;

  setUp(() => store = ReceiptStore(MemoryFileSystem(), '/receipts'));

  test('writes inside the root', () async {
    final path = await store.save('2026/lunch.pdf', Uint8List(4));
    expect(p.split(path), containsAllInOrder(['receipts', '2026', 'lunch.pdf']));
  });

  for (final attack in ['../secrets.txt', '/etc/passwd', 'a/../../x']) {
    test('refuses $attack', () {
      expect(() => store.save(attack, Uint8List(1)), throwsA(isA<UnsafePath>()));
    });
  }

  test('windows separators join correctly', () {
    final win = p.Context(style: p.Style.windows);
    expect(win.join(r'C:\data', 'a.txt'), r'C:\data\a.txt');
  });
}
```

`MemoryFileSystem(style: FileSystemStyle.windows)` runs the same file code with Windows paths.

## Flutter Apps

Flutter apps can't choose arbitrary folders. `path_provider` gives you `getApplicationDocumentsDirectory()` and `getTemporaryDirectory()`, and you join under those with `p.join`. Store relative paths in your database, because an iOS app's container path changes between installs and updates.

## Verify

```bash
dart analyze
dart test
grep -rn "'/'" lib | grep -v "p\.\|Uri\|http"     # hand-built separators left
```

Run CI on Windows if the code ships there; a `Context` test covers logic, but not real file behaviour such as case-insensitive names.

## Gotchas

1. **`FileSystemException: path not found` on Windows only.** Symptom: it works on macOS. Cause: a `/` was concatenated in. Fix: use `p.join`.
2. **`p.join` returned the second path.** Symptom: the root disappeared. Cause: the second argument is absolute. Fix: reject it, or strip the root.
3. **`..` escapes the folder.** Symptom: a file lands outside the store. Cause: no containment check. Fix: normalize, then `p.isWithin`.
4. **`%20` in a file name.** Symptom: a file isn't found. Cause: `uri.path` kept the encoding. Fix: use `p.fromUri`.
5. **A saved path stops working on iOS.** Symptom: images vanish after an update. Cause: an absolute container path was stored. Fix: store a relative path, and rebuild it at runtime.
6. **`p.isWithin` is false for the root itself.** Symptom: a check rejects the folder. Cause: it means strictly inside. Fix: compare with `p.equals` for that case.

## Anti-Patterns

- Trusting a name from a user, a server or an archive.
- Using `Directory.current` in a library. Pass paths in.
- Testing against real temp folders when `MemoryFileSystem` will do.
- Using `Uri.parse` on a Windows path. `C:` reads as a scheme.

## Pre-Merge Checklist

- [ ] No path is built by string concatenation
- [ ] Outside input is normalized, then checked with `p.isWithin`
- [ ] Path traversal cases are tested
- [ ] File code takes a `FileSystem`
- [ ] Stored paths are relative

## See Also

- `flutter-database` for files alongside local data
- `flutter-security` for sensitive file handling
- `dart-build-cli-tool` for tools working on paths
