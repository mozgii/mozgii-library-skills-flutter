---
title: The data layer of a feature
author: MOZGII, Inc.
lang: en
tags: [flutter, repository, dio, fpdart]
description: A repository implementation that turns every exception into a Failure, so nothing above the data layer sees a DioException.
---
# The Data Layer of a Feature

`data/` is the only layer that talks to the outside world, and the only one that sees an exception.

## Mapping Exceptions to Failure in a Repository

`AuthRepositoryImpl` catches what the transport throws and returns a `Left`. A 401 here means bad credentials, so it becomes `Failure.auth`. Any other status becomes `Failure.server`, and no response at all becomes `Failure.network`. A malformed body is a `FormatException` or a `TypeError` in `fromJson`, and an empty body leaves `response.data` null. All three are server faults, so each becomes a `Failure.server` and none crashes a screen.

```dart
// lib/features/auth/data/repositories/auth_repository_impl.dart
import 'package:dio/dio.dart';
import 'package:fpdart/fpdart.dart';
import 'package:injectable/injectable.dart';

import '../../../../core/errors/failure.dart';
import '../../domain/entities/user.dart';
import '../../domain/repositories/auth_repository.dart';
import '../models/user_model.dart';

@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl(this._dio);
  final Dio _dio;

  @override
  Future<Either<Failure, User>> signIn(String email, String password) async {
    try {
      final response = await _dio.post<Map<String, dynamic>>(
        '/auth/sign-in',
        data: {'email': email, 'password': password},
      );
      final data = response.data;
      if (data == null) return left(const Failure.server(200, 'Empty response'));
      return right(UserModel.fromJson(data).toEntity());
    } on DioException catch (e) {
      final status = e.response?.statusCode;
      if (status == 401) {
        return left(const Failure.auth('Wrong email or password'));
      }
      if (status != null) return left(Failure.server(status, 'Server error'));
      return left(Failure.network(e.message ?? 'Network error'));
    } on FormatException catch (e) {
      return left(Failure.server(200, 'Unreadable response: ${e.message}'));
    } on TypeError {
      return left(const Failure.server(200, 'Unreadable response'));
    }
  }
}
```
