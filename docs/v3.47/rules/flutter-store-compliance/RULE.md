---
title: "Flutter App Store & Play Store Compliance"
author: MOZGII, Inc.
lang: en
kind: rule
description: "App Store and Google Play submission compliance rules. Covers required iOS entitlements, Privacy Manifest (PrivacyInfo.xcprivacy), login and account deletion rules, screenshot requirements, Android Data Safety form, target SDK requirements, payment compliance with anti-steering and fees, and the full pre-submission checklist. Apply before every production release."
tags: ["flutter", "app-store", "play-store", "compliance", "privacy", "payments", "submission"]
---
# Flutter App Store & Play Store Compliance

Applies to all Flutter apps before any production release submission. Governs required entitlements, privacy manifests, data safety declarations, screenshot requirements, and payment compliance. Every item must pass before submission.

---

## iOS: Required Entitlements by Feature

| Feature used | Required entitlement / capability |
|---|---|
| Push notifications | Push Notifications capability |
| Apple Sign In | Sign In with Apple capability |
| Apple Pay | Apple Pay capability + merchant IDs |
| Face ID / Touch ID | `NSFaceIDUsageDescription` in `Info.plist` |
| Location (while in use) | `NSLocationWhenInUseUsageDescription` in `Info.plist` |
| Location (always) | `NSLocationAlwaysAndWhenInUseUsageDescription` in `Info.plist` |
| Camera | `NSCameraUsageDescription` in `Info.plist` |
| Microphone (voice AI) | `NSMicrophoneUsageDescription` in `Info.plist` |
| Photo library (read) | `NSPhotoLibraryUsageDescription` in `Info.plist` |
| Photo library (write) | `NSPhotoLibraryAddUsageDescription` in `Info.plist` |
| Background sync | Background Modes capability (fetch / remote notifications) |
| App Clips | App Clips capability + `NSAppClip` configuration |
| Health data | HealthKit capability + `NSHealthShareUsageDescription` |
| Bluetooth | `NSBluetoothAlwaysUsageDescription` in `Info.plist` |
| Contacts | `NSContactsUsageDescription` in `Info.plist` |

**Rule:** Every `NS*UsageDescription` string must describe the specific use case in plain language. Generic strings ("This app needs access") will be rejected by App Review.

---

## iOS Privacy Manifest

File: `ios/Runner/PrivacyInfo.xcprivacy`, added to the Runner target in Xcode.

A privacy manifest records the data types your app collects and the required reason APIs it uses. Apple's reasons requirement applies on iOS, iPadOS, tvOS, visionOS and watchOS. An upload that uses a required reason API with no declared reason gets an `ITMS-91053` notice, and App Store Connect has enforced that since May 1, 2024.

**Required reason APIs:** declare each category your own native code uses, under `NSPrivacyAccessedAPITypes`, with an approved reason. Each plugin should carry its own manifest for what it uses.

| Category (`NSPrivacyAccessedAPIType`) | Common trigger | Example reason |
|---|---|---|
| `NSPrivacyAccessedAPICategoryUserDefaults` | `NSUserDefaults`, the `shared_preferences` plugin | `CA92.1`: data only the app itself can read |
| `NSPrivacyAccessedAPICategoryFileTimestamp` | File creation or modification dates | `C617.1`: files inside the app or app group container |
| `NSPrivacyAccessedAPICategorySystemBootTime` | Uptime calculation | `35F9.1`: time elapsed between events in the app |
| `NSPrivacyAccessedAPICategoryDiskSpace` | Checking free space before a download | `E174.1`: whether there's room to write files |
| `NSPrivacyAccessedAPICategoryActiveKeyboards` | Reading active keyboards | `3EC4.1`: custom keyboard apps |

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
  <dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
      <string>CA92.1</string>
    </array>
  </dict>
</array>
```

**Declared data types (`NSPrivacyCollectedDataTypes`):** one dictionary per data type, with `NSPrivacyCollectedDataType`, `NSPrivacyCollectedDataTypeLinked`, `NSPrivacyCollectedDataTypeTracking` and `NSPrivacyCollectedDataTypePurposes`.

- `NSPrivacyCollectedDataTypeName`: if collecting a display name
- `NSPrivacyCollectedDataTypeEmailAddress`: if collecting email
- `NSPrivacyCollectedDataTypeCrashData`: if using Crashlytics
- `NSPrivacyCollectedDataTypePerformanceData`: if using Firebase Performance
- `NSPrivacyCollectedDataTypeOtherDiagnosticData`: if logging diagnostics

**Verify:** archive the app in Xcode, then right-click the archive in the Organizer and choose Generate Privacy Report. It shows the manifests of the app and its SDKs together, so a plugin with no manifest stands out. To check the file reached the bundle:

```bash
flutter build ipa --release
find build/ios/archive -name 'PrivacyInfo.xcprivacy' | head
```

---

## iOS Screenshot Requirements

| Display | Status |
|---|---|
| iPhone with Dynamic Island, medium | At least one screenshot, if the app runs on iPhone |
| iPhone with Dynamic Island, large, or iPhone with Face ID, large | One of the two sets is required, if the app runs on iPhone |
| iPad 13-inch | At least one screenshot, if the app runs on iPad |
| Other sizes | Optional: App Store Connect scales down from a larger set |

Apple's page defines these by pixel size, for example 1320 x 2868 for a large Dynamic Island display and 1284 x 2778 for a large Face ID display. Read the current sizes in App Store Connect Help.

**Rules:**
- At least 1 screenshot and up to 10 per device size and localization
- `.jpeg`, `.jpg` or `.png`, with no alpha channel or transparency
- Screenshots must match the current app UI: outdated ones are a rejection risk

---

## iOS App Review Rules

```
[ ] App does not crash on launch (test on real device, not Simulator)
[ ] No broken links or non-functional buttons
[ ] All features described in metadata are accessible without contacting support
[ ] Login-required apps: demo credentials provided in App Review notes
[ ] Guideline 4.8: an app using a third-party login for the primary account also offers an equivalent login service that limits data to name and email, lets users keep their email private, and doesn't collect interactions for ads without consent (Sign in with Apple qualifies; own-accounts-only, enterprise, education and government ID apps are exempt)
[ ] Guideline 5.1.1(v): an app that supports account creation offers account deletion inside the app
[ ] Subscription: Restore Purchases button visible on paywall and in Settings
[ ] Subscription: Terms of Service and Privacy Policy linked inside the app
[ ] No reference to other mobile platforms (Android, Google Play) in app UI or metadata
[ ] No use of private APIs (use entitlements for platform features, not private frameworks)
[ ] App Transport Security: NSAllowsArbitraryLoads must be false in production
```

---

## Android: Google Play Data Safety Form

Every app must complete the Data Safety section in Google Play Console before publishing. Inaccurate declarations are policy violations.

| Data type | Scenario | Declaration required |
|---|---|---|
| Name, Email | User account creation | Personal info: Account management |
| Precise location | Optional location feature | Location data: Precise location |
| Coarse location | Approximate location feature | Location data: Approximate location |
| Device or other IDs | Analytics, device fingerprint | Device or other IDs |
| Crash logs | Firebase Crashlytics | App info and performance: Crash logs |
| App diagnostics | Firebase Performance | App info and performance: Diagnostics |
| Purchase history | In-app purchases | Financial info: Purchase history |
| Search history | In-app search | App activity: Search history |
| Photos | Photo upload feature | Photos and videos: Photos |

**Encryption in transit:** Declare "Yes": all network calls must use TLS 1.2+.

**User data deletion:** Every app answers the Data deletion questions in this form. An app that lets users create an account must offer deletion inside the app and on a web page, and enter that link in the form. Google set December 7, 2023 as the deadline, extended to May 31, 2024 on request.

**Data that only transits (no storage, no sharing):** May declare "collected = No" only if the data is genuinely not retained on any server, even temporarily for processing.

---

## Apple Build and Minimum OS Requirements

Since April 28, 2026, the App Store accepts only uploads built in Xcode 26 or newer against an iOS 26 SDK. Apple's own floor is iOS 13 (September 9, 2026), but Flutter 3.47 raised its minimums to iOS 15 and macOS 12. The iOS 27 SDK requires the UIScene lifecycle, so migrate a custom `AppDelegate` before you move to it.

---

## Android Target SDK Requirements

```kotlin
// android/app/build.gradle.kts
android {
    compileSdk = 36

    defaultConfig {
        minSdk = 24      // Flutter's default floor since 3.35
        targetSdk = 36   // Android 16: required by Google Play for new apps and updates
    }
}
```

Google Play has required API 36 for new apps and app updates since August 31, 2026, with an extension available on request until November 1, 2026. Existing apps need API 35 to stay available to new users on newer Android versions. Flutter 3.47 defaults to compileSdk 36, targetSdk 36 and minSdk 24, and needs Java 17, Android Gradle Plugin 9.1.0 and Gradle 9.3.1.

**Rule:** `targetSdk` must meet Google Play's current requirement. Check https://developer.android.com/google/play/requirements/target-sdk for the current deadline.

**Verify before submission:**
```bash
flutter build appbundle --release
# bundletool reads an AAB directly; aapt's badging command expects an APK
bundletool dump manifest --bundle=build/app/outputs/bundle/release/app-release.aab | grep targetSdkVersion
```

---

## Payment Compliance

### Permitted payment methods by goods type

| Goods / service type | Required payment method |
|---|---|
| Digital goods (in-app content, premium features, virtual currency) | In-app purchase or Play Billing, unless an exception applies |
| Subscriptions to digital features | In-app purchase or Play Billing, directly or through RevenueCat |
| Physical goods (e-commerce, restaurant orders) | A method other than in-app purchase, such as Stripe. Apple's 3.1.3(e) requires this |
| Ride-hailing, food delivery, service bookings | A method other than in-app purchase |
| Real-time 1:1 services (tutoring, consultations) | Other methods allowed; one-to-many services use in-app purchase |

### Anti-Steering Rules

**Apple:** The rule depends on the storefront. Apps on the United States storefront may include buttons, external links and other calls to action to other purchase methods, and need no entitlement (see 3.1.3 and 3.1.1(a)). In specific regions, a StoreKit External Purchase Link Entitlement allows a link to a permitted destination. In every other storefront, apps and their metadata may not include such buttons, links or calls to action. Communicating with users outside the app is always allowed. Switch any external purchase link by storefront, not per build.

**Google:** Inside the app, pointing users at any way to pay besides Play Billing is forbidden unless a program applies. US users can be reached through the enrolled external content links and alternative billing programs, and India, South Korea and the EEA have their own programs. Outside the app, you're free to talk to users about other options.

### Store fees

Rates change and vary by storefront, so read Apple's and Google's pages before pricing. On 2026-10-31 they read:

| Store | Rates |
|---|---|
| App Store, standard | 30%; 15% for qualifying subscriptions after year one |
| App Store programs | 15% for Small Business (up to 1 million USD in proceeds), Video Partner, Mini Apps Partner and News Partner |
| App Store, other storefronts | China mainland 25% standard; EU 26% for in-app purchase, 15% for program participants |
| Play, in the EEA, UK, US, Australia and Japan | First $1M of annual earnings: 10% plus a 5% billing fee. Subscriptions: 10% plus 5%. Other sales: 20% plus 5% for new installs, 25% plus 5% for existing installs |
| Play, all other markets | 15% on the first $1M, 30% above it; 15% on auto-renewing subscriptions at any revenue |

### Required payment UI elements

```
[ ] Restore mechanism for restorable purchases: visible on paywall screen and in app Settings
[ ] Subscription management link: links to iOS subscription management
     (URL: https://apps.apple.com/account/subscriptions) or Play subscription management
[ ] Price displayed in local currency before purchase confirmation
[ ] Trial terms clearly stated (duration, what happens after trial ends)
[ ] Cancellation policy stated before purchase
```

---

## Pre-Submission Checklist

### iOS

```
[ ] PrivacyInfo.xcprivacy present and complete (all Required Reasons APIs declared)
[ ] All Info.plist NS*UsageDescription strings present for every permission used
[ ] NSFaceIDUsageDescription present (if any biometric authentication used)
[ ] Equivalent privacy login present where guideline 4.8 applies (Sign in with Apple qualifies)
[ ] Account deletion works inside the app, if accounts can be created
[ ] All required screenshots uploaded: iPhone medium and large display sets (+ iPad 13-inch if supported)
[ ] Demo account credentials in App Review notes (if login required to use app)
[ ] Privacy Policy URL set in App Store Connect
[ ] Terms of Service URL set in App Store Connect
[ ] App built with: flutter build ipa --release --obfuscate --split-debug-info=build/debug-symbols/ios
[ ] dSYM files uploaded to Firebase Crashlytics (via Fastlane or Xcode Organizer)
[ ] App Check debug provider absent from release build
[ ] NSAllowsArbitraryLoads = false in Info.plist
[ ] Restore Purchases button visible on paywall (if any IAP)
```

### Android

```
[ ] Data Safety form complete and accurate in Google Play Console
[ ] targetSdk is 36 or higher in android/app/build.gradle.kts
[ ] AAB built with: flutter build appbundle --release --obfuscate --split-debug-info=build/debug-symbols/android
[ ] R8 left on in the release build (the Flutter Gradle plugin enables it unless you pass --no-shrink)
[ ] Native debug symbols uploaded to Firebase Crashlytics
[ ] Play Integrity (App Check) enabled and enforced
[ ] Account deletion offered in the app and on a web page, with the link in the Data Safety form
[ ] No service account key or other real secret in the repo
[ ] App tested on real mid-range Android device (Samsung A-series or equivalent)
```

### Both platforms

```
[ ] No use of prohibited APIs (no UDID access, no background clipboard read on iOS 16+)
[ ] App does not crash on launch (tested on real physical device)
[ ] All navigation links and buttons are functional
[ ] Restore Purchases accessible (if any IAP present)
[ ] Analytics consent gate present and functional (GDPR compliance)
[ ] flutter-firebase rule: verify initialization order and credential file exclusions
[ ] flutter-security: pre-release hardening checklist passed
[ ] flutter-quality: coverage thresholds met, lint clean
[ ] App version and build number incremented from last submission
[ ] Release notes (What's New) written for this version
```
