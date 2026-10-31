---
title: "App Store and Google Play payment rules, fees and anti-steering"
author: MOZGII, Inc.
lang: en
tags: [flutter, payments, app-store, google-play, fees, anti-steering]
description: Which payment method each store requires, the commission tables read from Apple's and Google's pages on 2026-10-31, and the external link rules including the United States exceptions.
---
# Store Payment Rules and Fees

These rules were read from Apple's App Review Guidelines and Developer pages and from Play Console Help on 2026-10-31. Both stores changed terms during 2026, and rates depend on the storefront, so confirm against those pages before you price a product. This is a summary of store terms, not legal advice.

## Digital vs Physical Goods Rule

| Purchase type | Required method |
|---|---|
| Features, subscriptions, in-game currency, premium content | In-app purchase, or Play Billing, unless an exception below applies |
| Physical goods, or services used outside the app (Apple 3.1.3(e)) | A method other than in-app purchase, such as Apple Pay or card entry |
| Real-time person-to-person services (Apple 3.1.3(d)) | May use other methods; one-to-few and one-to-many must use in-app purchase |
| Content or subscriptions bought elsewhere, in a reader app (Apple 3.1.3(a)) | Access is allowed; magazines, newspapers, books, audio, music and video only |
| Multiplatform access (Apple 3.1.3(b)) | Allowed if the items are also sold as in-app purchases in the app |
| Sold to organizations for their staff or students (Apple 3.1.3(c)) | Allowed; consumer sales still use in-app purchase |
| Free companion to a paid web tool (Apple 3.1.3(f)) | Allowed for VoIP, cloud storage, email and web hosting, with no purchase inside the app |

Google Play requires its billing for in-app purchases of digital goods, including currency, subscriptions and unlocks. Physical goods and services, digital goods that can't be used in the app, in-app gift cards, peer-to-peer payments and 1:1 online services are outside that requirement. When an item mixes digital and physical content, billing applies if it holds more digital than physical content or is marketed as digital. Consumption-only apps, with nothing to buy inside, aren't subject to it.

## Apple App Store Commission Rates

| Rate | Applies to |
|---|---|
| 30% | Standard rate on digital goods and services |
| 15% | App Store Small Business Program, Video Partner, Mini Apps Partner and News Partner programs |
| 15% | Qualifying subscriptions after their first year |

The Small Business Program needs no more than 1 million USD in proceeds in the previous calendar year and the current one, counted across associated accounts. Passing the threshold moves future sales to the standard rate.

Storefronts differ. On the China mainland, standard transactions have been 25% since March 15, 2026, and Small Business and Mini Apps Partner transactions are lower. In the EU, Apple's business terms list 26% for in-app purchase and 15% for program participants, and alternative terms carry their own rates, including 10% for Small Business participants and for subscriptions after their first year. Read `developer.apple.com/support/apps-in-the-eu` for the current table.

## Google Play Service Fees

Google's newer fees roll out by region, and the install date picks the fee. The EEA, UK and US rolled out on June 30, 2026, and Australia and Japan on September 30, 2026. An install is "new" when the user's first install or first update from Play falls on or after that date.

These apply in Australia, the EEA, Japan, the UK and the US. In the table, the added 5% is the billing fee:

| Fee type | Auto-renewing subscriptions | Other, new installs | Other, existing installs |
|---|---|---|---|
| First $1M of annual earnings | 10% + 5% | 10% + 5% | 10% + 5% |
| Standard | 10% + 5% | 20% + 5% | 25% + 5%, or 20% for external web links |
| Play Games Level Up and Apps Experience programs | 10% + 5% | 15% + 5% | 20% + 5%, or 15% for external web links |

The billing fee applies when the user pays through Google Play Billing, or on the first purchase of a paid app.

All other markets, until the update reaches them:

| Fee type | Rate |
|---|---|
| Enrolled in the 15% tier, first $1M per year | 15% |
| Enrolled in the 15% tier, above $1M | 30% |
| Auto-renewing subscriptions, at any revenue | 15% |

In South Korea and India, a developer offering an alternative billing system pays the Play rate reduced by 4%.

## Anti-Steering Rules on the App Store

Anti-steering rules limit what an app may say inside itself about paying elsewhere. Apple's rule now depends on the storefront.

| Storefront | Inside the app |
|---|---|
| United States | Buttons, external links and other calls to action to other purchase methods are allowed. No entitlement is required (see 3.1.3 and 3.1.1(a)) |
| Regions with a StoreKit External Purchase Link Entitlement | A link to a permitted destination is allowed with that entitlement, within its agreement. It may state that items cost less there |
| All others | Neither the app nor its metadata can carry a button, external link or call to action that points to another purchase method |

Developers can always contact their own users outside the app about other ways to pay. A reader app can request the External Link Account Entitlement, which lets it link to account creation or management on its site. In the United States it doesn't need one. A link used deceptively can remove the app and the developer account.

A Flutter build runs in every storefront, so an external purchase link must be switched by storefront rather than shipped globally. Don't infer the storefront from the device locale; use StoreKit's storefront value.

## Anti-Steering Rules on Google Play

Inside the app, a developer can't steer users to any way to pay besides Play billing, including through direct links or language encouraging purchase elsewhere, unless a program applies. Outside the app, you can talk to users freely. Links to administrative pages such as account management or a privacy policy are fine if they don't lead to a prohibited payment method. Consumption-only apps may mention purchase options without a direct link.

| Program | Who it's for | Terms |
|---|---|---|
| US external content links | US users, territories included | Enrollment and API integration required. In-app item fees follow the tiers above. Linked app installs cost $3.65 for games and $2.85 for apps |
| US alternative billing | Users in the United States | Offered under the US programs; check the program page |
| India and South Korea alternative billing | Users there | Offered beside Play Billing, at 4% less |
| EEA programs | Users in the EEA, under the Digital Markets Act | Alternative billing and external links, subject to program rules |

For the external content links program, Google says enrolled developers report transactions and pay fees starting October 1, 2026, and the deadline for reporting downloads moved to February 1, 2027. Check the program page for the current date.

## Required Subscription Information

Apple's 3.1.1 requires a restore mechanism for restorable purchases, and says purchased credits or in-game currency may not expire. Subscription terms belong on the paywall, before purchase. Link to your Terms of Service and Privacy Policy from the app.
