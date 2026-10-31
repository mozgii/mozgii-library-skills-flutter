---
name: flutter-payments
description: Adds payments to a Flutter app - Stripe PaymentSheet with 3DS, Apple Pay and Google Pay, RevenueCat subscriptions, server-side receipt validation with the App Store Server API and Google Play Developer API, and the store rules on fees and external links. Use when adding a checkout, an in-app purchase or subscription, a paywall, a restore button, or when asking which payment method the stores allow.
metadata:
  tags:
  - flutter
  - payments
  - stripe
  - apple-pay
  - google-pay
  - revenuecat
  - iap
  - subscriptions
  - pci
---
# Flutter Payments

The payment layer for a production Flutter app. The stores decide which payment method you may use, and the server decides what a customer has paid for. The Flutter side stays thin: it starts a payment and shows the result, and raw card data never touches Dart code.

## Step 0: Read Your Versions

```bash
flutter --version
grep -nE "^\s+(flutter_stripe|pay|purchases_flutter|url_launcher|dio):" pubspec.yaml
dart pub outdated --no-dev-dependencies
```

Pins were read from pub.dev on 2026-10-30 for Flutter 3.47.9 and Dart 3.13.7, and the examples were analyzed against them. `purchases_flutter` 11 uses Play Billing Library 9.1.0 and needs Kotlin 2.1.0 and `compileSdk` 35; a second billing plugin in that app has to work with Play Billing 9.1.0 too. Store rules and fees change often: the figures here were read from Apple's and Google's pages on 2026-10-31, so check them before you price anything. Everything here was compiled, not run on a device, so run each purchase in the stores' sandboxes on real hardware. On `package:flutter/material.dart`, swap the `material_ui` import in the UI references, or run `dart fix --apply --code=migrate_design_widgets`. The examples also import `flutter_riverpod: ^3.4.3`, `material_ui: ^1.9.0`, `url_launcher: ^6.3.3`.

## Payment Method Decision Table

| What is sold | Use | Why the others lose here |
|---|---|---|
| Digital goods, features, subscriptions | In-app purchase through RevenueCat | Both stores require their billing here outside the exceptions in `references/store-rules-and-fees.md` |
| Physical goods, or services used outside the app | Stripe with Apple Pay and Google Pay | Apple's guideline 3.1.3(e) requires a method other than in-app purchase |
| Both | RevenueCat and Stripe in parallel | One provider can't serve both legally |
| Real-time 1:1 services (tutoring, consultations) | Stripe | Apple's 3.1.3(d) and Google's rules allow it; one-to-many services need in-app purchase |

| Method | Platforms | PCI scope |
|---|---|---|
| Apple Pay | iOS | Apple handles card data |
| Google Pay | Android | Google handles card data |
| Stripe PaymentSheet | iOS, Android, Web | Card data goes straight to Stripe |
| RevenueCat | iOS, Android | The stores handle it |

## Stripe Integration with flutter_stripe

PaymentSheet is Stripe's prebuilt UI. It collects the card natively, runs 3D Secure when the bank asks for it, and offers Apple Pay and Google Pay from one screen, so most apps never need separate buttons.

```yaml
dependencies:
  flutter_stripe: ^14.2.0
  dio: ^5.11.1
```

### Stripe Android Setup

Android needs two changes the Flutter template doesn't have, because the Stripe SDK uses AppCompat components and the support fragment manager for its sheets. `flutter_stripe` supports API 21+, and Flutter's own default `minSdk` is 24. The R8 rules are in `references/native-pay-buttons.md`.

```kotlin
// android/app/src/main/kotlin/com/example/app/MainActivity.kt
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity : FlutterFragmentActivity()
```

```xml
<!-- android/app/src/main/res/values/styles.xml, and values-night -->
<style name="LaunchTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowBackground">@drawable/launch_background</item>
</style>
<style name="NormalTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowBackground">?android:colorBackground</item>
</style>
```

### Stripe iOS Setup

No `AppDelegate` code is needed. `flutter_stripe` registers its own app and scene delegates and forwards redirect URLs to the Stripe SDK, leaving other links alone. Set the deployment target to iOS 15 or later: the `stripe_ios` 14.2.0 podspec asks for 15.0, the same floor as a new Flutter project. For redirect methods, register a URL scheme in `CFBundleURLTypes`, and use that same scheme for `Stripe.urlScheme` and `returnURL`.

### Stripe Initialization

A publishable key is safe in the app; the secret key never leaves your server. Google Pay's test mode follows the key, so a test key can't reach production Google Pay by accident.

```dart
// lib/core/payments/stripe_setup.dart
import 'package:flutter_stripe/flutter_stripe.dart';

const stripePublishableKey = String.fromEnvironment('STRIPE_PUBLISHABLE_KEY');
const paymentReturnScheme = 'com.example.app';

bool get stripeTestMode => stripePublishableKey.startsWith('pk_test_');

Future<void> initStripe() async {
  Stripe.publishableKey = stripePublishableKey;
  Stripe.merchantIdentifier = 'merchant.com.example.app'; // For Apple Pay.
  Stripe.urlScheme = paymentReturnScheme;
  await Stripe.instance.applySettings();
}
```

### Worked Example: Checkout with PaymentSheet

Your backend creates the PaymentIntent, because only it knows the price. It returns the client secret, and for a known customer also a customer ID and ephemeral key. Those two are optional and belong together. Never send empty strings: a guest checkout passes null for both.

```dart
// lib/features/payment/data/stripe_checkout.dart
import 'package:dio/dio.dart';
import 'package:flutter_stripe/flutter_stripe.dart';
import 'package:yourapp/core/payments/stripe_setup.dart';

final class CheckoutSession {
  const CheckoutSession({
    required this.clientSecret,
    this.customerId,
    this.ephemeralKey,
  });

  factory CheckoutSession.fromJson(Map<String, dynamic> json) =>
      CheckoutSession(
        clientSecret: json['client_secret'] as String,
        customerId: json['customer_id'] as String?,
        ephemeralKey: json['ephemeral_key'] as String?,
      );

  final String clientSecret;
  final String? customerId;
  final String? ephemeralKey;
}

sealed class CheckoutResult {
  const CheckoutResult();
}

/// Paid on Stripe's side. The webhook still decides what the user receives.
final class CheckoutPaid extends CheckoutResult {
  const CheckoutPaid();
}

final class CheckoutCancelled extends CheckoutResult {
  const CheckoutCancelled();
}

final class CheckoutFailed extends CheckoutResult {
  const CheckoutFailed(this.message);

  final String message;
}

class StripeCheckout {
  StripeCheckout(this._dio);

  final Dio _dio;

  Future<CheckoutResult> pay({
    required String productId,
    required String merchantName,
  }) async {
    try {
      final response = await _dio.post<Map<String, dynamic>>(
        '/payments/create-intent',
        data: {'product_id': productId}, // The server sets amount and currency.
      );
      final session = CheckoutSession.fromJson(response.data!);

      await Stripe.instance.initPaymentSheet(
        paymentSheetParameters: SetupPaymentSheetParameters(
          paymentIntentClientSecret: session.clientSecret,
          customerId: session.customerId,
          customerEphemeralKeySecret: session.ephemeralKey,
          merchantDisplayName: merchantName,
          returnURL: '$paymentReturnScheme://stripe-redirect',
          applePay: const PaymentSheetApplePay(merchantCountryCode: 'US'),
          googlePay: PaymentSheetGooglePay(
            merchantCountryCode: 'US',
            testEnv: stripeTestMode,
          ),
        ),
      );
      await Stripe.instance.presentPaymentSheet();
      return const CheckoutPaid();
    } on StripeException catch (e) {
      return e.error.code == FailureCode.Canceled
          ? const CheckoutCancelled()
          : CheckoutFailed(e.error.localizedMessage ?? 'Payment failed');
    } on DioException {
      return const CheckoutFailed('Network error. You were not charged.');
    }
  }
}
```

Refunds start on the server through the Stripe API. Never expose a refund endpoint the client can call.

## Apple Pay and Google Pay Buttons

PaymentSheet already offers both wallets. Reach for the `pay` package only when the design needs a standalone wallet button. `references/native-pay-buttons.md` has the configuration and the confirmation step.

## RevenueCat In-App Purchases and Subscriptions

RevenueCat wraps StoreKit and Play Billing and validates each purchase with the store. Your backend's user ID goes in as `appUserID`, so purchases follow the account across devices.

```yaml
dependencies:
  purchases_flutter: ^11.0.0
```

**`purchases_flutter` 11 won't resolve on CocoaPods until after November 7, 2026.** Its 11.0.0 changelog says the `PurchasesHybridCommon` 20.0.0 pod is held back by the CocoaPods trunk freeze of November 1 to 7. Until that pod is out, build iOS and macOS with Swift Package Manager or stay on 10.x. Trunk turns read-only on December 2, 2026, and RevenueCat plans to drop CocoaPods in a later major, so Swift Package Manager is the lasting choice.

```dart
// lib/core/iap/purchases_service.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:purchases_flutter/purchases_flutter.dart';

sealed class PurchaseOutcome {
  const PurchaseOutcome();
}

final class PurchaseDone extends PurchaseOutcome {
  const PurchaseDone();
}

/// The user backed out: show nothing.
final class PurchaseCancelled extends PurchaseOutcome {
  const PurchaseCancelled();
}

/// Ask to Buy or a slow payment method: no access yet, no error either.
final class PurchasePending extends PurchaseOutcome {
  const PurchasePending();
}

final class PurchaseFailed extends PurchaseOutcome {
  const PurchaseFailed(this.message);

  final String message;
}

class PurchasesService {
  Future<void> configure({required String userId}) async {
    final apiKey = defaultTargetPlatform == TargetPlatform.iOS
        ? const String.fromEnvironment('RC_IOS_API_KEY')
        : const String.fromEnvironment('RC_ANDROID_API_KEY');
    if (kDebugMode) await Purchases.setLogLevel(LogLevel.debug);
    await Purchases.configure(
      PurchasesConfiguration(apiKey)..appUserID = userId,
    );
  }

  Future<List<Package>> fetchPackages() async {
    final offerings = await Purchases.getOfferings();
    return offerings.current?.availablePackages ?? const [];
  }

  Future<PurchaseOutcome> buy(Package package) async {
    try {
      // purchase() has returned PurchaseResult since version 9.
      await Purchases.purchase(PurchaseParams.package(package));
      return const PurchaseDone();
    } on PlatformException catch (e) {
      return switch (PurchasesErrorHelper.getErrorCode(e)) {
        PurchasesErrorCode.purchaseCancelledError => const PurchaseCancelled(),
        PurchasesErrorCode.paymentPendingError => const PurchasePending(),
        _ => PurchaseFailed(e.message ?? 'Purchase failed'),
      };
    }
  }

  Future<CustomerInfo> restore() => Purchases.restorePurchases();

  Future<void> signOut() async {
    await Purchases.logOut();
  }
}
```

## Which Entitlement Check to Trust

Your server is the source of truth for anything it delivers: premium API calls, AI credits, downloads, content. The client's `CustomerInfo` is a display hint, and a modified app can fake any local check. So the app reads entitlements from your backend, and `CustomerInfo` only shapes the UI for features that run entirely on the device.

| Protecting | Check on | Why |
|---|---|---|
| A server action or paid content | Your backend's entitlement record | A patched client can't change it |
| A paywall, badge or button state | Your backend's answer, cached locally | The same answer everywhere |
| A device-only feature, while offline | `CustomerInfo` from the SDK | It's cached for offline use, and the worst case is one patched copy |

A webhook can land after the purchase sheet closes, so don't make the user wait for it. After `PurchaseDone`, call an endpoint such as `POST /billing/sync`, which makes your server ask RevenueCat directly with `GET /v1/subscribers/{app_user_id}` and update its record. The webhook then keeps that record current.

```dart
// lib/core/iap/entitlement_api.dart
import 'package:dio/dio.dart';

class EntitlementApi {
  EntitlementApi(this._dio);

  final Dio _dio;

  /// Call after a purchase or restore: the server asks RevenueCat first.
  Future<Set<String>> syncAfterPurchase() async {
    final response = await _dio.post<Map<String, dynamic>>('/billing/sync');
    final active = response.data!['active'] as List<dynamic>;
    return active.cast<String>().toSet();
  }
}
```

## Server-Side Receipt Validation

Never trust the client's report of a successful purchase. Entitlement changes only after your server hears from Stripe, RevenueCat or the store.

| Provider | Your server listens to | Then |
|---|---|---|
| RevenueCat | Webhooks: `INITIAL_PURCHASE`, `RENEWAL`, `NON_RENEWING_PURCHASE`, `CANCELLATION`, `EXPIRATION` | Grant on purchase and renewal, including one-time products; end access on `EXPIRATION`, because `CANCELLATION` alone only turns off renewal. `CANCELLATION` with `cancel_reason: CUSTOMER_SUPPORT` is a refund, so recheck the status and revoke at once |
| Stripe | Signed webhooks: `payment_intent.succeeded`, `payment_intent.payment_failed`, `charge.refunded` | Verify `Stripe-Signature` on the raw body, and skip event IDs you've processed |
| Apple, direct | App Store Server API and Server Notifications V2 | Verify the signed transaction; `/verifyReceipt` is deprecated |
| Google, direct | Play Developer API `purchases.subscriptionsv2.get` or `purchases.products.get` | Acknowledge within three days, or Google refunds the purchase |

`references/server-receipt-validation.md` covers each path, with the fraud cases to guard against. Never read `PaymentIntent.status` in the app as final confirmation.

## Store Rules, Fees and Anti-Steering

The stores' fee tables and external-link rules differ by storefront and changed during 2026. `references/store-rules-and-fees.md` has both stores' tables, and the anti-steering rules with their United States exceptions.

Required on any paywall:

| Element | Required by |
|---|---|
| A restore mechanism for restorable purchases | Apple guideline 3.1.1 |
| Price and billing period before purchase | Apple and Google |
| Trial terms before purchase | Apple and Google |
| A link to manage the subscription | Apple and Google |
| Cancellation instructions | Google Play |

`references/paywall-ui.md` has the paywall, the subscription card, the links to subscription management on each store, and a failed payment dialog.

## Verify

```dart
// test/core/iap/entitlement_rule_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:yourapp/features/payment/data/stripe_checkout.dart';

void main() {
  test('a guest checkout carries no customer values', () {
    final session = CheckoutSession.fromJson({'client_secret': 'pi_1_secret_2'});
    expect(session.customerId, isNull);
    expect(session.ephemeralKey, isNull);
  });
}
```

Run each case on real builds: Stripe test card `4242 4242 4242 4242`, and a 3D Secure card from Stripe's testing page. Use a sandbox Apple account and a Play license tester. Then confirm by hand that a cancelled sheet shows no error, a pending purchase grants nothing, a restore on a second device works, and a refund or expiration removes access on the server.

```bash
stripe listen --forward-to localhost:4242/webhook    # watch the webhooks arrive
stripe trigger payment_intent.succeeded
grep -rnE "sk_live_|sk_test_" lib/                   # expect nothing
```

## Gotchas

1. **PaymentSheet fails on Android only.** Symptom: `is not a subclass FlutterFragmentActivity`, or a theme error, when the sheet opens. Cause: `MainActivity` extends `FlutterActivity`, or the theme isn't AppCompat. Fix: use `FlutterFragmentActivity` and a `Theme.AppCompat` parent, then rebuild: hot reload won't apply it.
2. **`Const variables must be initialized with a constant value`.** Symptom: a compile error on `PaymentConfiguration`. Cause: `fromJsonString` isn't a const constructor. Fix: use `final`.
3. **PaymentSheet fails with an empty customer.** Symptom: an init error once a customer is passed. Cause: empty strings for `customerId` and the ephemeral key. Fix: pass real values from the backend, or null for a guest.
4. **Google Pay is stuck in test mode.** Symptom: production shows the test wallet. Cause: `bool.fromEnvironment('DEBUG')`, a define no build sets. Fix: derive `testEnv` from the key prefix.
5. **The user paid and has no access.** Symptom: premium stays locked for a few seconds. Cause: the webhook hasn't landed. Fix: call a server sync after the purchase.
6. **The cancel button shows an error.** Symptom: "Purchase failed" after backing out. Cause: cancellation arrives as an exception. Fix: match `FailureCode.Canceled` and `purchaseCancelledError`.
7. **App Review rejects the app.** Symptom: rejected under 3.1.1. Cause: digital goods sold through Stripe outside an exception. Fix: use in-app purchase.
8. **Entitlements go empty near a Play renewal.** Symptom: a paying user sees the paywall briefly. Cause: RevenueCat documents this gap around renewal. Fix: rely on your server's record, with a short grace period.

## Anti-Patterns

- Setting price or currency in the client. The server decides what's charged.
- Granting access in `onPaymentResult` or after `presentPaymentSheet`. Wait for server confirmation.
- Treating an Apple Pay or Google Pay token as proof of payment.
- Revoking access on every `CANCELLATION`. The user paid until the period ends, unless `cancel_reason` is `CUSTOMER_SUPPORT`, which marks a refund.
- Hardcoding prices in the paywall. Show the store's localized `priceString`.

## Pre-Merge Checklist

- [ ] No secret key or raw card data appears in the app
- [ ] `FlutterFragmentActivity` and AppCompat themes are in place
- [ ] `customerId` and the ephemeral key are real values or null
- [ ] The server sets amounts and confirms payment by signed webhook
- [ ] Webhook handlers are idempotent
- [ ] The app reads entitlements from the backend
- [ ] Restore works, and the paywall shows price, period and trial terms
- [ ] Cancelled and pending purchases are handled without an error
- [ ] External purchase links match the storefront rules

## See Also

- `flutter-auth` for the user ID that purchases attach to
- `flutter-security` for pinning and release hardening
- `flutter-network` for Dio setup
- `flutter-analytics-firebase` for purchase events
