---
title: "Apple Pay and Google Pay buttons with the pay package"
author: MOZGII, Inc.
lang: en
tags: [flutter, payments, apple-pay, google-pay, stripe]
description: Standalone wallet buttons with the pay package - entitlements, payment configurations, the button widget, and confirming the wallet token with Stripe.
---
# Native Wallet Buttons with the pay Package

PaymentSheet already includes both wallets, so use the `pay` package only when a design calls for a wallet button outside it. The token a wallet returns is not a payment: your backend or Stripe still has to confirm it.

```yaml
dependencies:
  pay: ^3.3.0
```

## Apple Pay Entitlement

In Xcode, open Signing & Capabilities, add Apple Pay and select your Merchant ID. That writes this entitlement into `Runner.entitlements`:

```xml
<key>com.apple.developer.in-app-payments</key>
<array>
  <string>merchant.com.example.app</string>
</array>
```

## Wallet Payment Configurations

`PaymentConfiguration.fromJsonString` is a normal factory, so these are `final`, not `const`. The README recommends loading configuration from a remote source when you can, which lets you change it without a release. Set `environment` to `PRODUCTION` and fill in your Google Pay merchant ID before launch.

```dart
// lib/shared/payments/wallet_configs.dart
import 'package:pay/pay.dart';

final applePayConfig = PaymentConfiguration.fromJsonString('''
{
  "provider": "apple_pay",
  "data": {
    "merchantIdentifier": "merchant.com.example.app",
    "displayName": "Example",
    "merchantCapabilities": ["3DS", "debit", "credit"],
    "supportedNetworks": ["amex", "discover", "masterCard", "visa"],
    "countryCode": "US",
    "currencyCode": "USD",
    "requiredBillingContactFields": [],
    "requiredShippingContactFields": []
  }
}
''');

final googlePayConfig = PaymentConfiguration.fromJsonString('''
{
  "provider": "google_pay",
  "data": {
    "environment": "TEST",
    "apiVersion": 2,
    "apiVersionMinor": 0,
    "allowedPaymentMethods": [{
      "type": "CARD",
      "parameters": {
        "allowedAuthMethods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
        "allowedCardNetworks": ["AMEX", "DISCOVER", "MASTERCARD", "VISA"]
      },
      "tokenizationSpecification": {
        "type": "PAYMENT_GATEWAY",
        "parameters": {
          "gateway": "stripe",
          "stripe:version": "2024-06-20",
          "stripe:publishableKey": "pk_test_replace_me"
        }
      }
    }],
    "merchantInfo": {
      "merchantId": "YOUR_GOOGLE_PAY_MERCHANT_ID",
      "merchantName": "Example"
    },
    "transactionInfo": {
      "totalPriceStatus": "FINAL",
      "totalPrice": "9.99",
      "currencyCode": "USD",
      "countryCode": "US"
    }
  }
}
''');
```

## Wallet Button Widget

```dart
// lib/shared/payments/wallet_button.dart
import 'package:flutter/foundation.dart';
import 'package:material_ui/material_ui.dart';
import 'package:pay/pay.dart';
import 'package:yourapp/shared/payments/wallet_configs.dart';

class WalletButton extends StatelessWidget {
  const WalletButton({
    super.key,
    required this.amount,
    required this.onResult,
    required this.onError,
  });

  final String amount;
  final void Function(Map<String, dynamic> result) onResult;
  final void Function(Object? error) onError;

  @override
  Widget build(BuildContext context) {
    final items = [
      PaymentItem(
        label: 'Total',
        amount: amount,
        status: PaymentItemStatus.final_price,
      ),
    ];
    const spinner = Center(child: CircularProgressIndicator());

    if (defaultTargetPlatform == TargetPlatform.iOS) {
      return ApplePayButton(
        paymentConfiguration: applePayConfig,
        paymentItems: items,
        onPaymentResult: onResult,
        onError: onError,
        loadingIndicator: spinner,
        style: ApplePayButtonStyle.black,
        type: ApplePayButtonType.buy,
      );
    }
    return GooglePayButton(
      paymentConfiguration: googlePayConfig,
      paymentItems: items,
      onPaymentResult: onResult,
      onError: onError,
      loadingIndicator: spinner,
      type: GooglePayButtonType.buy,
    );
  }
}
```

## Confirming a Google Pay Token with Stripe

The result holds a tokenized card. Create the PaymentIntent on your server, then confirm with the token, so the server still controls the amount. The webhook grants access, as with any other payment.

```dart
// lib/shared/payments/confirm_google_pay.dart
import 'dart:convert';

import 'package:flutter_stripe/flutter_stripe.dart';

Future<void> confirmGooglePay(
  Map<String, dynamic> result,
  String clientSecret,
) async {
  final paymentData = result['paymentMethodData'] as Map<String, dynamic>;
  final tokenization = paymentData['tokenizationData'] as Map<String, dynamic>;
  final token = jsonDecode(tokenization['token'] as String) as Map<String, dynamic>;

  await Stripe.instance.confirmPayment(
    paymentIntentClientSecret: clientSecret,
    data: PaymentMethodParams.cardFromToken(
      paymentMethodData: PaymentMethodDataCardFromToken(
        token: token['id'] as String,
      ),
    ),
  );
}
```

## Stripe R8 Rules

`flutter_stripe` lists these rules for Android release builds. Add them whichever Stripe UI you use.

```proguard
# android/app/proguard-rules.pro
-dontwarn com.stripe.android.pushProvisioning.**
-dontwarn com.google.android.gms.tapandpay.**
-keep class com.stripe.** { *; }
```
