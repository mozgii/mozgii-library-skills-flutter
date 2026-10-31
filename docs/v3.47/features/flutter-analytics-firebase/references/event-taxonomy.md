---
title: "Firebase Analytics event taxonomy for Flutter"
author: MOZGII, Inc.
lang: en
tags: [flutter, firebase, analytics, events]
description: A complete typed AnalyticsService with auth, AI, purchase, error and engagement events, a single consent check, and no free text in any parameter.
---
# Analytics Event Taxonomy

The service below is the whole event surface of an app: one method per event, so a privacy review reads one file. Every call passes through `_log`, which checks consent. Parameters hold codes, IDs of things and counts, and never text a person typed. Booleans go as 1 or 0, because Analytics takes strings and numbers.

```dart
// lib/core/analytics/analytics_service.dart
import 'package:firebase_analytics/firebase_analytics.dart';

enum PurchaseFailureCode { cancelled, declined, network, unknown }

enum AuthFailureCode { invalidCredentials, rateLimited, network, unknown }

class AnalyticsService {
  AnalyticsService(this._analytics, this._consented);

  final FirebaseAnalytics _analytics;
  final bool Function() _consented;

  Future<void> _log(Future<void> Function() send) async {
    if (_consented()) await send();
  }

  // Auth
  Future<void> logSignIn({required String method}) =>
      _log(() => _analytics.logLogin(loginMethod: method));

  Future<void> logSignUp({required String method}) =>
      _log(() => _analytics.logSignUp(signUpMethod: method));

  Future<void> logSignOut() =>
      _log(() => _analytics.logEvent(name: 'auth_sign_out'));

  Future<void> logBiometricPrompt({required bool success}) => _log(
        () => _analytics.logEvent(
          name: 'auth_biometric_prompt',
          parameters: {'success': success ? 1 : 0},
        ),
      );

  Future<void> logPinVerified({required bool success}) => _log(
        () => _analytics.logEvent(
          name: 'auth_pin_verified',
          parameters: {'success': success ? 1 : 0},
        ),
      );

  // AI
  Future<void> logAiChatStarted({required String modelId}) => _log(
        () => _analytics.logEvent(
          name: 'ai_chat_started',
          parameters: {'model_id': modelId},
        ),
      );

  Future<void> logAiMessageSent({
    required String modelId,
    required bool hasAttachment,
  }) =>
      _log(
        () => _analytics.logEvent(
          name: 'ai_message_sent',
          parameters: {
            'model_id': modelId,
            'has_attachment': hasAttachment ? 1 : 0,
          },
        ),
      );

  Future<void> logAiVoiceUsed({required String modelId}) => _log(
        () => _analytics.logEvent(
          name: 'ai_voice_used',
          parameters: {'model_id': modelId},
        ),
      );

  Future<void> logAiModelLoaded({
    required String modelId,
    required int loadTimeMs,
  }) =>
      _log(
        () => _analytics.logEvent(
          name: 'ai_model_loaded',
          parameters: {'model_id': modelId, 'load_time_ms': loadTimeMs},
        ),
      );

  // Purchase
  Future<void> logPurchaseStarted({
    required String productId,
    required String currency,
    required double value,
  }) =>
      _log(
        () => _analytics.logBeginCheckout(
          currency: currency,
          value: value,
          items: [AnalyticsEventItem(itemId: productId)],
        ),
      );

  Future<void> logPurchaseCompleted({
    required String productId,
    required String transactionId,
    required String currency,
    required double value,
  }) =>
      _log(
        () => _analytics.logPurchase(
          currency: currency,
          value: value,
          transactionId: transactionId,
          items: [AnalyticsEventItem(itemId: productId)],
        ),
      );

  Future<void> logPurchaseFailed({
    required String productId,
    required PurchaseFailureCode code,
  }) =>
      _log(
        () => _analytics.logEvent(
          name: 'purchase_failed',
          parameters: {'product_id': productId, 'reason': code.name},
        ),
      );

  Future<void> logSubscriptionViewed({required String planId}) => _log(
        () => _analytics.logEvent(
          name: 'purchase_subscription_viewed',
          parameters: {'plan_id': planId},
        ),
      );

  // Errors
  /// [routeTemplate] is `/orders/:id`, never the real path.
  Future<void> logApiError({
    required String routeTemplate,
    required int statusCode,
  }) =>
      _log(
        () => _analytics.logEvent(
          name: 'error_api',
          parameters: {
            'endpoint': routeTemplate.length > 100
                ? routeTemplate.substring(0, 100)
                : routeTemplate,
            'status_code': statusCode,
          },
        ),
      );

  Future<void> logAuthError({required AuthFailureCode code}) => _log(
        () => _analytics.logEvent(
          name: 'error_auth',
          parameters: {'reason': code.name},
        ),
      );

  Future<void> logSyncError({required String feature}) => _log(
        () => _analytics.logEvent(
          name: 'error_sync',
          parameters: {'feature': feature},
        ),
      );

  // Engagement
  Future<void> logFeatureUsed({required String featureName}) => _log(
        () => _analytics.logEvent(
          name: 'engagement_feature_used',
          parameters: {'feature_name': featureName},
        ),
      );

  /// Never the query text: people type names and numbers into search boxes.
  Future<void> logSearchPerformed({
    required String category,
    required int resultCount,
  }) =>
      _log(
        () => _analytics.logEvent(
          name: 'search_performed',
          parameters: {'category': category, 'result_count': resultCount},
        ),
      );
}
```

`logPurchaseCompleted` sends a transaction ID, which is an ID of a purchase, not a person, and analytics needs it to deduplicate revenue.
