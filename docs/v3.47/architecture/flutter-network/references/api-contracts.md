---
title: API response contracts
author: MOZGII, Inc.
lang: en
tags: [flutter, api, freezed, repository]
description: A sealed ApiResponse type, the standard error envelope and how to parse it, and the rules that keep raw HTTP inside the data layer.
---
# API Response Contracts

A contract says what a repository may return and what a server may answer. Writing both down keeps raw HTTP inside the data layer.

## Sealed ApiResponse Type

`ApiResponse<T>` gives a screen one value to switch over, with every state covered by the compiler.

```dart
// lib/core/network/api_response.dart
import 'package:freezed_annotation/freezed_annotation.dart';

import '../errors/failure.dart';

part 'api_response.freezed.dart';

@freezed
sealed class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse.success(T data) = ApiResponseSuccess<T>;
  const factory ApiResponse.error(Failure failure) = ApiResponseError<T>;
  const factory ApiResponse.loading() = ApiResponseLoading<T>;
}
```

## Standard Error Envelope

All API errors must conform to this envelope on the backend.

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Email is invalid",
  "details": [
    { "field": "email", "reason": "Must be a valid email address" }
  ]
}
```

Parse it in the data source, with `explicitToJson` so the nested details serialise.

```dart
// lib/core/network/api_error.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_error.freezed.dart';
part 'api_error.g.dart';

@freezed
abstract class ApiError with _$ApiError {
  @JsonSerializable(explicitToJson: true)
  const factory ApiError({
    required String code,
    required String message,
    @Default(<ApiErrorDetail>[]) List<ApiErrorDetail> details,
  }) = _ApiError;

  factory ApiError.fromJson(Map<String, dynamic> json) =>
      _$ApiErrorFromJson(json);
}

@freezed
abstract class ApiErrorDetail with _$ApiErrorDetail {
  const factory ApiErrorDetail({
    required String field,
    required String reason,
  }) = _ApiErrorDetail;

  factory ApiErrorDetail.fromJson(Map<String, dynamic> json) =>
      _$ApiErrorDetailFromJson(json);
}
```

## Repository Contract Rules

- Repository methods return `Future<Either<Failure, T>>`, never a raw HTTP response and never `Map<String, dynamic>`.
- DTOs (models) exist only in `data/models/`. They convert to domain entities through a `.toEntity()` method.
- Domain entities in `domain/entities/` have zero JSON knowledge.
- Never return `List<dynamic>` from a repository: always `List<ConcreteEntity>`.
- Failure types are sealed, so every consumer must handle all cases.
