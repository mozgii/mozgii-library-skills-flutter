---
title: Response caching and request deduplication
author: MOZGII, Inc.
lang: en
tags: [flutter, dio, caching, performance]
description: HTTP caching for Dio with a Hive store, stale data on server errors, and sharing one in-flight request between callers.
---
# Response Caching and Request Deduplication

Caching avoids a request, and deduplication shares one. Together they stop a screen with three widgets from making the same call three times.

```yaml
dependencies:
  dio_cache_interceptor: ^4.0.7
  http_cache_hive_store: ^5.1.1   # replaces the discontinued dio_cache_interceptor_hive_store
```

## Cache Interceptor Setup

`CachePolicy.request` follows the server's cache headers: cache first, validated with an ETag. `hitCacheOnNetworkFailure` and `hitCacheOnErrorCodes` serve stale data for up to `maxStale` when the server is down. Keep that list to server errors, and never add 401 or 403: those mean the person may not see this data, and a cached copy would show it anyway.

```dart
// lib/core/network/interceptors/cache_interceptor.dart
import 'package:dio_cache_interceptor/dio_cache_interceptor.dart';
import 'package:http_cache_hive_store/http_cache_hive_store.dart';

DioCacheInterceptor buildCacheInterceptor(String cacheDir) {
  final store = HiveCacheStore(cacheDir);
  return DioCacheInterceptor(
    options: CacheOptions(
      store: store,
      policy: CachePolicy.request, // Cache-first, validate with ETag
      hitCacheOnErrorCodes: const [500, 502, 503, 504], // never 401 or 403
      hitCacheOnNetworkFailure: true,
      maxStale: const Duration(hours: 24),
      priority: CachePriority.normal,
    ),
  );
}
```

The cache is a file on disk. Clear it on sign-out, so a second account on the device never reads the first account's responses.

## Request Deduplication

`RequestDeduplicator` keeps one `Future` per key. A second caller gets the first caller's `Future`, and the entry goes away when it completes, whether it worked or not, so a failed request can be tried again. Use it for reads only: two identical writes may both be meant.

```dart
// lib/core/network/request_deduplicator.dart
import 'package:injectable/injectable.dart';

@lazySingleton
class RequestDeduplicator {
  final _inflight = <String, Future<Object?>>{};

  Future<T> deduplicate<T>(String key, Future<T> Function() factory) {
    final existing = _inflight[key];
    if (existing != null) return existing as Future<T>;

    final future = factory().whenComplete(() => _inflight.remove(key));
    _inflight[key] = future;
    return future;
  }
}
```

In a data source it reads `_deduplicator.deduplicate('user_$id', () => _dio.get('/users/$id'))`.
