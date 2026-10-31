---
title: "Flutter paywall UI with RevenueCat"
author: MOZGII, Inc.
lang: en
tags: [flutter, payments, paywall, revenuecat, ui]
description: A complete paywall with loading, empty and error states, the subscription card with its billing period, Terms and Privacy links, restore, sync failures and a failed payment dialog, built from theme values and RevenueCat's store data.
---
# Paywall UI

A paywall shows what the store says: the localized `priceString`, the trial and the billing period. Apple's guideline 3.1.2 also wants working links to your Terms and Privacy Policy on it. It uses theme values, so it's easy to fit into any design system. After a purchase it asks the server what the user owns, because the server is the source of truth.

Three cases must never end in silence. A failed restore shows its error. A sync that can't reach the server says so. And a person who just paid, where the sync finds no entitlement, is told the payment went through and what to do next, because the webhook can arrive after the sync.

## Paywall Screen

```dart
// lib/features/subscription/paywall_page.dart
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';
import 'package:purchases_flutter/purchases_flutter.dart';
import 'package:yourapp/features/subscription/manage_subscription.dart';
import 'package:yourapp/features/subscription/purchases_service.dart';
import 'package:yourapp/features/subscription/subscription_card.dart';

final purchasesServiceProvider = Provider<PurchasesService>(
  (ref) => PurchasesService(),
);

/// Override this with a call to your backend's sync endpoint.
final syncEntitlementsProvider = Provider<Future<Set<String>> Function()>(
  (ref) => throw UnimplementedError('Provide the entitlement sync'),
);

final offeringsProvider = FutureProvider.autoDispose<List<Package>>(
  (ref) => ref.watch(purchasesServiceProvider).fetchPackages(),
);

class PaywallPage extends ConsumerStatefulWidget {
  const PaywallPage({super.key});

  @override
  ConsumerState<PaywallPage> createState() => _PaywallPageState();
}

class _PaywallPageState extends ConsumerState<PaywallPage> {
  bool _busy = false;

  void _show(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }

  Future<void> _purchase(Package package) async {
    setState(() => _busy = true);
    final outcome = await ref.read(purchasesServiceProvider).buy(package);
    if (!mounted) return;
    switch (outcome) {
      case PurchaseDone():
        await _sync(
          whenEmpty:
              'Your payment went through, but your upgrade has not reached '
              'your account yet. Tap Restore in a minute, or contact support.',
        );
      case PurchaseCancelled():
        break;
      case PurchasePending():
        _show('Your purchase is waiting for approval.');
      case PurchaseFailed(:final message):
        _show(message);
    }
    if (mounted) setState(() => _busy = false);
  }

  Future<void> _restore() async {
    setState(() => _busy = true);
    try {
      await ref.read(purchasesServiceProvider).restore();
      await _sync(whenEmpty: 'No purchases found.');
    } on PlatformException catch (e) {
      if (mounted) _show(e.message ?? 'Restore failed. Try again.');
    } finally {
      if (mounted) setState(() => _busy = false);
    }
  }

  Future<void> _sync({required String whenEmpty}) async {
    try {
      final active = await ref.read(syncEntitlementsProvider)();
      if (!mounted) return;
      if (active.contains('premium')) {
        Navigator.of(context).pop(true);
      } else {
        _show(whenEmpty);
      }
    } on Exception {
      if (mounted) {
        _show('We could not check your account. Tap Restore to try again.');
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    final offerings = ref.watch(offeringsProvider);
    return Scaffold(
      appBar: AppBar(
        title: const Text('Upgrade to Pro'),
        actions: [
          TextButton(
            onPressed: _busy ? null : _restore,
            child: const Text('Restore'),
          ),
        ],
      ),
      body: switch (offerings) {
        AsyncData(value: final packages) when packages.isEmpty => const Center(
            child: Text('No plans are available right now.'),
          ),
        AsyncData(value: final packages) => ListView(
            padding: const EdgeInsets.all(16),
            children: [
              for (final package in packages)
                SubscriptionCard(
                  package: package,
                  busy: _busy,
                  onSelect: () => _purchase(package),
                ),
              Center(
                child: TextButton(
                  onPressed: openSubscriptionManagement,
                  child: const Text('Manage subscription'),
                ),
              ),
              const Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  TextButton(
                    onPressed: openTerms,
                    child: Text('Terms of Use'),
                  ),
                  TextButton(
                    onPressed: openPrivacyPolicy,
                    child: Text('Privacy Policy'),
                  ),
                ],
              ),
            ],
          ),
        AsyncError() => Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Text('Could not load plans.'),
                TextButton(
                  onPressed: () => ref.invalidate(offeringsProvider),
                  child: const Text('Try again'),
                ),
              ],
            ),
          ),
        _ => const Center(child: CircularProgressIndicator()),
      },
    );
  }
}
```

## Subscription Card

An introductory price is only a free trial when its price is 0, so the card says which one it is. Real terms need the period length and what the user pays afterward. `subscriptionPeriod` is an ISO 8601 duration such as `P1M`, and it's null for a one-time product, so the price line reads `$9.99 per month` for a subscription and the bare price otherwise.

```dart
// lib/features/subscription/subscription_card.dart
import 'package:material_ui/material_ui.dart';
import 'package:purchases_flutter/purchases_flutter.dart';

class SubscriptionCard extends StatelessWidget {
  const SubscriptionCard({
    super.key,
    required this.package,
    required this.busy,
    required this.onSelect,
  });

  final Package package;
  final bool busy;
  final VoidCallback onSelect;

  /// `P1M` becomes `per month`, `P3M` becomes `every 3 months`.
  String? _billingPeriod(String? iso) {
    final match = RegExp(r'^P(\d+)([DWMY])$').firstMatch(iso ?? '');
    if (match == null) return null;
    final count = int.parse(match[1]!);
    final unit = const {
      'D': 'day',
      'W': 'week',
      'M': 'month',
      'Y': 'year',
    }[match[2]!]!;
    return count == 1 ? 'per $unit' : 'every $count ${unit}s';
  }

  String? _trialLine(IntroductoryPrice? intro) {
    if (intro == null) return null;
    final length = '${intro.periodNumberOfUnits} ${intro.periodUnit.name}';
    return intro.price == 0
        ? 'Free for $length'
        : '${intro.priceString} for $length';
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final product = package.storeProduct;
    final trial = _trialLine(product.introductoryPrice);
    final period = _billingPeriod(product.subscriptionPeriod);
    final highlighted = package.packageType == PackageType.annual;

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(16),
        side: highlighted
            ? BorderSide(color: theme.colorScheme.primary, width: 2)
            : BorderSide.none,
      ),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(product.title, style: theme.textTheme.titleMedium),
                  if (trial != null)
                    Text(
                      trial,
                      style: theme.textTheme.bodySmall?.copyWith(
                        color: theme.colorScheme.primary,
                      ),
                    ),
                  Text(
                    period == null
                        ? product.priceString
                        : '${product.priceString} $period',
                    style: theme.textTheme.bodyMedium,
                  ),
                ],
              ),
            ),
            FilledButton(
              onPressed: busy ? null : onSelect,
              child: const Text('Select'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Purchases Service for the Paywall

The paywall imports the service from its feature folder. This is the same class as in the skill, trimmed to what the screen calls.

```dart
// lib/features/subscription/purchases_service.dart
import 'package:flutter/services.dart';
import 'package:purchases_flutter/purchases_flutter.dart';

sealed class PurchaseOutcome {
  const PurchaseOutcome();
}

final class PurchaseDone extends PurchaseOutcome {
  const PurchaseDone();
}

final class PurchaseCancelled extends PurchaseOutcome {
  const PurchaseCancelled();
}

final class PurchasePending extends PurchaseOutcome {
  const PurchasePending();
}

final class PurchaseFailed extends PurchaseOutcome {
  const PurchaseFailed(this.message);

  final String message;
}

class PurchasesService {
  Future<List<Package>> fetchPackages() async {
    final offerings = await Purchases.getOfferings();
    return offerings.current?.availablePackages ?? const [];
  }

  Future<PurchaseOutcome> buy(Package package) async {
    try {
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
}
```

## Manage Subscription, Terms and Privacy Links

Replace both legal URLs with your own pages.

```dart
// lib/features/subscription/manage_subscription.dart
import 'package:flutter/foundation.dart';
import 'package:url_launcher/url_launcher.dart';

Future<void> _open(String url) =>
    launchUrl(Uri.parse(url), mode: LaunchMode.externalApplication);

Future<void> openSubscriptionManagement() => _open(
      defaultTargetPlatform == TargetPlatform.iOS
          ? 'https://apps.apple.com/account/subscriptions'
          : 'https://play.google.com/store/account/subscriptions',
    );

Future<void> openTerms() => _open('https://example.com/terms');

Future<void> openPrivacyPolicy() => _open('https://example.com/privacy');
```

## Failed Payment Dialog

A failed payment shows a non-blocking dialog with a retry, and it never blames the user.

```dart
// lib/features/subscription/payment_failed_dialog.dart
import 'package:material_ui/material_ui.dart';

Future<void> showPaymentFailedDialog(
  BuildContext context, {
  required String reason,
  required VoidCallback onRetry,
}) {
  return showDialog<void>(
    context: context,
    builder: (context) => AlertDialog(
      title: const Text('Payment failed'),
      content: Text(reason),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancel'),
        ),
        FilledButton(
          onPressed: () {
            Navigator.pop(context);
            onRetry();
          },
          child: const Text('Try again'),
        ),
      ],
    ),
  );
}
```
