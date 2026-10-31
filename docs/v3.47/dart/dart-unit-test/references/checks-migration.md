---
title: "Migrating expect matchers to package:checks"
author: MOZGII, Inc.
lang: en
tags: [dart, testing, checks]
description: A lookup table from package:matcher to package:checks, with the import changes and the mistakes a migration makes.
---
# Migrating expect Matchers to package:checks

Migrate a file at a time. `package:matcher` comes along with `package:test`, so nothing needs removing unless a pubspec lists it directly.

## Imports for Each Stage

```bash
dart pub add dev:checks
```

A fully migrated file swaps `package:test/test.dart` for `package:test/scaffolding.dart` and `package:checks/checks.dart`. While a file is half done, add `package:test/expect.dart` so the old `expect` still resolves, and remove it at the end.

## Lookup Table for Common Matchers

| Before | After | Note |
|---|---|---|
| `expect(a, b)`, `expect(a, equals(b))` | `check(a).equals(b)` | Use `deepEquals` for collections |
| `isA<T>()` | `check(a).isA<T>()` | Chains on |
| `same(b)` | `check(a).identicalTo(b)` | Identity |
| `hasLength(n)` | `check(a).length.equals(n)` | String, Map, Iterable |
| `contains(x)` | `check(a).contains(x)` | Maps use `containsKey` |
| `anyElement(m)` | `check(a).any((e) => e.equals(1))` | Takes a condition |
| `everyElement(m)` | `check(a).every((e) => e.isGreaterThan(0))` | Takes a condition |
| `isNot(m)` | `check(a).not((it) => it.equals(5))` | Takes a condition |
| `startsWith(s)`, `endsWith(s)` | `check(a).startsWith(s)`, `check(a).endsWith(s)` | Strings |
| `isEmpty`, `isNotEmpty` | `check(a).isEmpty()`, `check(a).isNotEmpty()` | They are methods here |
| `isNull`, `isNotNull` | `check(a).isNull()`, `check(a).isNotNull()` | |
| `isTrue`, `isFalse` | `check(a).isTrue()`, `check(a).isFalse()` | The subject can't be a `bool?` |
| `completion(m)` | `await check(f).completes((v) => v.equals(1))` | Await it |
| `throwsA(isA<E>())` | `await check(f).throws<E>()` | Also awaited |
| `emits(v)` | `await check(queue).emits((e) => e.equals(v))` | On a `StreamQueue` |
| `emitsThrough(v)` | `await check(queue).emitsThrough((e) => e.equals(v))` | On a `StreamQueue` |
| `stringContainsInOrder(l)` | `check(a).containsInOrder(l)` | Strings |
| `pairwiseCompare(...)` | `check(a).pairwiseMatches(...)` | |

## Mistakes a Migration Makes

A missing `await` on an async check lets the test pass with no assertion run. The `unawaited_futures` lint catches it. Comparing lists with `equals` fails on identity, so use `deepEquals`. A synchronous throw still needs a closure: `check(() => parse('x')).throws<FormatException>()`. And a stream needs a `StreamQueue` when you make several checks in order.

Find what's left with `grep -rn "expect(" test/`, and confirm with `dart test`.
