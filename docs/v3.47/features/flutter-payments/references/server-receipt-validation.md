---
title: "Server-side purchase validation for Flutter apps"
author: MOZGII, Inc.
lang: en
tags: [flutter, payments, server, app-store-server-api, play-developer-api, webhooks]
description: What your backend does for RevenueCat, Stripe, Apple and Google purchases - webhook handling, the App Store Server API that replaces verifyReceipt, Play purchase checks, and the fraud cases each one stops.
---
# Server-Side Purchase Validation

Purchase validation happens on your backend, because a client can lie about anything. The app sends an identifier; the server asks the provider and records the answer.

## RevenueCat Webhooks

RevenueCat validates receipts with Apple and Google. Add your backend URL in the dashboard under Integrations, Webhooks. Handle events by `id`, because retries reuse it, which makes duplicates easy to skip.

| Event | Means | Your server does |
|---|---|---|
| `INITIAL_PURCHASE`, `RENEWAL`, `UNCANCELLATION` | Paid or resumed | Grant or extend |
| `NON_RENEWING_PURCHASE` | A purchase that won't renew, such as a one-time product | Grant it; no `RENEWAL` or `EXPIRATION` follows on its own |
| `CANCELLATION`, `cancel_reason` not `CUSTOMER_SUPPORT` | Renewal turned off | Note it; access continues to the period end |
| `CANCELLATION`, `cancel_reason: CUSTOMER_SUPPORT` | A refund | Read the current status, then revoke what was refunded |
| `EXPIRATION` | Access is over | Remove access |
| `BILLING_ISSUE` | A payment failed | Warn the user |
| `SUBSCRIPTION_PAUSED` | Pause scheduled | Don't revoke; wait for `EXPIRATION` with the pause reason |
| `PRODUCT_CHANGE` | Plan change | Update the tier at the change |
| `TRANSFER` | Purchases moved between App User IDs | Move access from each id in `transferred_from` to each in `transferred_to` |
| `REFUND_REVERSED` | A refund was undone (App Store only) | Read the current status, then restore what the refund took away |

RevenueCat sends `CANCELLATION` for a subscription refund only when the latest period is refunded, and a refund can leave auto-renewal on, so the status call below decides, not the event alone. Its docs tie `CUSTOMER_SUPPORT` to subscription refunds and don't name a `cancel_reason` for refunded one-time purchases, so treat any `CANCELLATION` on a non-renewing product as a refund.

`TRANSFER` is the one event with no `app_user_id`, so a handler that reads that field first drops it: use `transferred_from` and `transferred_to`, both lists. The webhook fires for the receiving user alone, although both customers' histories list the event.

To read the state on demand, call `GET https://api.revenuecat.com/v1/subscribers/{app_user_id}` with an `Authorization: Bearer` header. Do this right after a purchase, so the user doesn't wait for the webhook. RevenueCat notes that unsubscribe and billing status can lag by hours without Platform Server Notifications, so turn those on.

## Stripe Webhooks

Register the endpoint in Stripe's Workbench, Webhooks tab, and select `payment_intent.succeeded`, `payment_intent.payment_failed` and `charge.refunded`. The endpoint verifies the `Stripe-Signature` header against the raw body with the `whsec_` secret, using Stripe's library. A framework that parses or rewrites the body breaks the check.

Return a 2xx status before any slow work, and queue the rest. Stripe retries for up to three days in live mode, doesn't guarantee order, and can deliver an event twice, so record processed event IDs. The library rejects old timestamps with a default tolerance of five minutes, which stops replays. Don't set the tolerance to 0: that turns the check off.

## App Store Server API

`/verifyReceipt` is deprecated, and Apple deprecated receipts with it. New integrations use the App Store Server API and App Store Server Notifications V2. The app sends a `transactionId`; your server fetches the signed record from Apple.

| Task | Endpoint |
|---|---|
| Check one purchase | `GET /inApps/v1/transactions/{transactionId}` (Get Transaction Info) |
| Check a subscription | `GET /inApps/v1/subscriptions/{transactionId}` (Get All Subscription Statuses) |
| Hosts | `https://api.storekit.itunes.apple.com`, and `https://api.storekit-sandbox.itunes.apple.com` for sandbox |

Requests carry a short-lived JWT signed with a private key from App Store Connect. Responses are JWS values: verify the signature and certificate chain up to Apple's root before you trust the payload, and prefer Apple's App Store Server Library to hand-written checks. Then check that `bundleId` is yours and that `appAccountToken`, if you set one, matches the signed-in user.

Register a Server Notifications V2 URL so renewals, refunds and expirations reach you without polling.

## Google Play Developer API

The app sends the `purchaseToken` to your backend, which calls `purchases.subscriptionsv2.get` for subscriptions or `purchases.products.get` for one-time products.

- Grant only in the `PURCHASED` state, never for `PENDING`.
- Keep every token. `purchaseToken` is globally unique, so it works as a primary key; `orderId` doesn't, because some purchases have none.
- When `linkedPurchaseToken` is set, revoke the entitlement granted to the old token, so two users can't share one purchase.
- Acknowledge right after granting, from the server: a purchase not acknowledged within three days is refunded automatically.
- For an illegitimate purchase, don't grant or acknowledge. Revoke and refund it with `purchases.subscriptionsv2:revoke` or `Orders:refund`.

Set up Real-time Developer Notifications so renewals and cancellations reach you.

## Fraud Patterns to Guard Against

| Attack | Defense |
|---|---|
| Replay of an old purchase | Store `transactionId` and `purchaseToken`, and refuse a second use |
| A faked purchase on a modified device | Validate on the server; add App Check for your own endpoints |
| A receipt from another app | Check `bundleId` or `packageName` |
| A receipt forwarded to another user | Bind each purchase to the authenticated user |
| A forged webhook | Verify `Stripe-Signature`, or the RevenueCat authorization or signature you configured |
| Duplicate delivery | Process each event ID once |
