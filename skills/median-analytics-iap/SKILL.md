---
name: median-analytics-iap
description: "Integrate analytics, crash reporting, and monetization into a Median.co app — Adjust, AppsFlyer, Firebase Analytics, Firebase Crashlytics, Apple StoreKit and Google Play Billing in-app purchases (median.iap.*), RevenueCat paywalls, deep linking, offline downloads. Use when a Median app needs event tracking and attribution, per-user crash logs, purchase and subscription flows, or universal links/app links."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/analytics
---

# Median Analytics, Attribution, IAP & Deep Links

Add mobile analytics and attribution SDKs (Adjust, AppsFlyer, Firebase Analytics), crash reporting (Firebase Crashlytics), in-app purchases (Apple StoreKit / Google Play Billing, plus the RevenueCat alternative), deep linking, and offline downloads to an existing web app running inside a Median.co app. The JS Bridge is invoked via the global `median` object (or `import Median from "median-js-bridge"` where the docs show the NPM form); calls should be made after the bridge library is ready — define `median_library_ready()` on the page (the app calls it once the library initializes; if the library initialized before the page defined it, call it manually inside `if (window.median) { ... }`), or use `Median.onReady()` with the NPM package. Analytics plugins wrap native SDKs — they need App Studio setup (tokens, config files) plus a save + rebuild before the bridge calls work.

## When to Use

Use this skill when the task involves:
- Adjust event tracking and attribution (`median.adjust.*`)
- AppsFlyer user association, events, conversion data, and deep-link results (bridge + NPM listeners)
- Firebase Analytics consent, collection toggle, user identity, events, screens, eCommerce
- In-app purchases end-to-end (Apple StoreKit / Google Play Billing): product listing (`median.iap.info()` / `median_info_ready`), purchases (`median.iap.purchase()`), verification and fulfillment, purchase history (`median.iap.purchases()`), restore, subscription management, and the Google Billing 4→6 `legacyMode` migration
- Firebase Crashlytics runtime control — enable/disable log collection, WebView console errors, Android toast errors, per-user crash IDs (`median.firebaseCrashlytics.*`)
- RevenueCat as a managed IAP backend with native paywall modals (`median.revenueCat.*`)
- Configuring deep linking: iOS Universal Links (AASA), Android App Links (assetlinks.json), custom URL schemes
- The offline download manager (`median.downloads.*`)
- Understanding plugin tier licensing/trial behavior or appConfig.json rules that gate all of the above

Don't use for:
- Push notifications or native auth (OneSignal, Clerk, Auth0, social login, passkeys) → use the `median-push-auth` skill
- Braze, Klaviyo, Meta App Events, Branch, Intent Edge — plugins exist but their APIs are outside this skill's scope (see https://docs.median.co/docs/analytics.md)

## Prerequisites

- **Median app** with the relevant plugins enabled in App Studio → Native Plugins (each change requires save + rebuild):
  - **Adjust**: App Token (Adjust dashboard AppView → app → App information → App details); Environment (**Production**/Development — set Production before publishing); Initialization (Automatic/Manual); StoreKit Ad Network Attribution (iOS).
  - **AppsFlyer**: Android Dev Key, Apple Dev Key, Apple App ID in Native Plugins > AppsFlyer > Settings. Requires AppsFlyer plugin 1.4.0 (iOS) / 1.3.0 (Android) for the conversion/deep-link APIs; they are NOT yet in the NPM package.
  - **Firebase Analytics**: upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) under Build & Deploy > Google Services; set default consent (Follow EU Consent Policy / Deny / Allow); enable the plugin.
  - **In-App Purchases**: products created in App Store Connect (save the shared secret) / Google Play Console (tiers = base plans of one subscription); a `productsUrl` JSON file hosted on your site (Apple: flat `products` array; Google: `inappProducts` + `subProducts` sections); Apple adds optional `postUrl` for server-side verification; Google has a `legacyMode` setting (Billing Library 4 compatibility). Full detail in `references/iap.md`.
  - **Firebase Crashlytics**: Firebase project with Crashlytics enabled; upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) under Build & Deploy > Google Services (same gate as Firebase Analytics). Settings: `requestOptIn` (consent prompt on first launch), `webErrorLogsEnabled`, `toastErrorLogsEnabled` (Android only).
  - **RevenueCat**: enable the plugin in Native Plugins (no config fields — initialization happens at runtime via `median.revenueCat.configure({ apiKey, appUserID })`); apps + products configured in a RevenueCat account.
- **Attribution accounts**: Adjust dashboard (event tokens are created there, not Median), AppsFlyer account.
- **Deep linking**: control of the target domain(s) to host `.well-known` verification files; iOS Associated Domains entitlement; Android app signing certificate SHA-256 fingerprint.
- **License tier**: Native Plugins availability depends on the Median license tier — some plugins are Business/Enterprise only (details in Pitfalls).

## Quick Reference

| Task | Call |
| --- | --- |
| Adjust: init (Manual mode only) | `median.adjust.intialize(true \| false)` |
| Adjust: track event (with revenue) | `const e = new AdjustEvent("abc123"); e.setRevenue(0.01, "EUR"); median.adjust.trackEvent(e)` |
| Adjust: read attribution | `median.adjust.attributionInfo()` |
| AppsFlyer: associate user | `await median.appsflyer.setCustomerUserId("user123")` |
| AppsFlyer: log event | `await median.appsflyer.logEvent("purchase", { value: 99.99, currency: "USD" })` |
| AppsFlyer: conversion data (cache read) | `await median.appsflyer.getConversionData()` |
| AppsFlyer: deep-link result (cache read) | `await median.appsflyer.getDeepLinkResult()` |
| AppsFlyer: NPM listeners | `Median.appsflyer.conversionData.addListener(fn)` / `.deeplinkResult.addListener(fn)` / `.sdkStart.addListener(fn)` |
| Firebase: set consent | `median.firebaseAnalytics.event.setConsent({ adStorage, analyticsStorage, adUserData, adPersonalization })` |
| Firebase: toggle collection | `median.firebaseAnalytics.event.collection({ enabled: false })` |
| Firebase: identify user | `median.firebaseAnalytics.event.setUser({ ID: STRING })` |
| Firebase: user property | `median.firebaseAnalytics.event.setUserProperty({ key, value })` |
| Firebase: default params | `median.firebaseAnalytics.event.defaultEventParameters(data)` |
| Firebase: log event | `median.firebaseAnalytics.event.logEvent({ event, data })` |
| Firebase: log screen | `median.firebaseAnalytics.event.logScreen({ screen })` |
| Firebase: eCommerce | `median.firebaseAnalytics.event.addToCart({ data, currency, price, quantity })` (also viewItem, addToWishlist, removeFromCart) |
| Crashlytics: toggle log collection | `median.firebaseCrashlytics.enable(true \| false)` |
| Crashlytics: error log toggles | `median.firebaseCrashlytics.webErrorLogsEnabled(b)` (WebView console errors) / `.toastErrorLogsEnabled(b)` (Android only) |
| Crashlytics: per-user crash logs | `median.firebaseCrashlytics.setUserId("user_123")` / `median.firebaseCrashlytics.unsetUserId()` |
| IAP: list products | `await median.iap.info()` or define `median_info_ready(productsData)` (called by the app on page load) |
| IAP: purchase | `median.iap.purchase({ productID })` — Google subs add `offerToken`; upgrades/downgrades add `previousPurchaseToken` + `prorationMode`/`replacementMode` |
| IAP: purchase history | `await median.iap.purchases()` or define `median_iap_purchases(purchasesData)` (called on load and after purchases) |
| IAP: restore (Apple) | `median.iap.restorePurchases()` |
| IAP: manage subscriptions (Google) | `median.iap.manageSubscription({ productID })` / `median.iap.manageAllSubscriptions()` |
| RevenueCat: present paywall | `median.revenueCat.presentPaywall()` (also `configure`, `getOfferings`, `purchase`, `restorePurchases`) |
| Deep link verification files | host AASA at `/apple-app-site-association`; assetlinks at `/.well-known/assetlinks.json` |
| Offline download | `median.downloads.init({ callback })` → `downloadFile({ url, title })` → `showUI()` |

## How It Works

### Adjust

NOTE ON SPELLING: the docs consistently spell the init function `intialize` (missing 'i') — that is verbatim docs, not a typo introduced here. Use `median.adjust.intialize(...)`.

```javascript
// Manual init; optional boolean enables/disables SKAN (iOS only, ignored on Android)
median.adjust.intialize(true | false);

const adjustEvent = new AdjustEvent("abc123");   // event token from Adjust dashboard
adjustEvent.setRevenue(0.01, "EUR");
median.adjust.trackEvent(adjustEvent);

median.adjust.attributionInfo();   // attribution stored by Adjust for the current user/install
```

Only call `intialize` when App Studio Initialization is set to Manual. Event tokens are created in the Adjust dashboard, not Median. `setRevenue(amount, currency)` needs a 3-letter currency code. Organic installs may return organic/limited attribution data. Test on real devices.

### AppsFlyer

Promise form (library) and NPM form (`import Median from "median-js-bridge";` → `Median.appsflyer.*`):

```javascript
const { success } = await median.appsflyer.setCustomerUserId("user123");
const { success } = await median.appsflyer.logEvent("purchase", { value: 99.99, currency: "USD" });

const conversion = await median.appsflyer.getConversionData();
const deepLink = await median.appsflyer.getDeepLinkResult();
```

Response shapes — success `{ "success": true, "data": { "af_status": "Organic", "is_first_launch": false } }`; nothing cached `{ "success": false, "data": null }`. Deep-link payloads are platform-divergent (iOS returns fields at root level; Android 1.4.0+ nests them under `data`; no deep link: Android `{ "success": false, "data": null, "error": { "message": "not_found" } }`, iOS `{}`) — read `deeplinkValue` defensively rather than branching on the shape. Full verbatim payloads, global callbacks (`median_appsflyer_cd_success`, `median_appsflyer_cd`, `median_appsflyer_deeplink_result`), and NPM listeners (`conversionData.addListener`, `deeplinkResult.addListener`, `sdkStart.addListener` → `removeListener(id)`) are in `references/analytics.md`. `af_status` is "Organic" or "Non-organic"; iOS 14.5+ requires ATT (configurable ATT prompt delay in the plugin). `getConversionData`/`getDeepLinkResult` are cache reads — they never wait for the SDK, so on a cold launch they can resolve with `data: null`.

### Firebase Analytics

```javascript
median.firebaseAnalytics.event.setConsent({
  adStorage: true | false, analyticsStorage: true | false,
  adUserData: true | false, adPersonalization: true | false,
});

median.firebaseAnalytics.event.collection({ enabled: BOOLEAN });

median.firebaseAnalytics.event.setUser({ ID: STRING });
median.firebaseAnalytics.event.setUserProperty({ key: STRING, value: STRING });

median.firebaseAnalytics.event.defaultEventParameters(data); // object of key-value pairs → bundle params

median.firebaseAnalytics.event.logEvent({ event: STRING, data: OBJECT });
median.firebaseAnalytics.event.logScreen({ screen: STRING });

// eCommerce (currency = 3-letter uppercase code; data = encoded ProductItem map)
median.firebaseAnalytics.event.viewItem({ data: JsonProductItem, currency: STRING, price: FLOAT });
median.firebaseAnalytics.event.addToWishlist({ data: JsonProductItem, currency: STRING, price: FLOAT, quantity: INTEGER });
median.firebaseAnalytics.event.addToCart({ data: JsonProductItem, currency: STRING, price: FLOAT, quantity: INTEGER });
median.firebaseAnalytics.event.removeFromCart({ data: JsonProductItem, currency: STRING, price: FLOAT, quantity: INTEGER });
```

ProductItem fields (all optional; recommend at least `item_id` or `item_name`): `item_id`, `item_name`, `item_category`, `item_variant`, `item_brand`, `item_list_name`, `item_list_id`, `price`. Full parameter table is in `references/analytics.md`. Firebase auto-logs `screen_view`, `session_start`, `first_open_time` by default.

### Firebase Crashlytics

Same Google Services gate as Firebase Analytics (upload `google-services.json` + `GoogleService-Info.plist`, enable the plugin, save + rebuild). Plugin settings: `requestOptIn` (consent prompt shown on first launch after installation), `webErrorLogsEnabled`, `toastErrorLogsEnabled` (Android only).

```javascript
median.firebaseCrashlytics.enable(true);                // runtime on/off for log collection
median.firebaseCrashlytics.webErrorLogsEnabled(true);   // WebView console errors
median.firebaseCrashlytics.toastErrorLogsEnabled(true); // Android toast errors (e.g. SSL warnings)
median.firebaseCrashlytics.setUserId("user_123");       // per-user crash logs — persists between launches
median.firebaseCrashlytics.unsetUserId();               // cleared only by this or uninstall/reinstall
```

Demo: https://median.dev/crashlytics — full parameter table in `references/analytics.md`.

*(docs quirk: the official Crashlytics troubleshooting section says to call functions after the `deviceready` event — a Cordova concept absent from Median; use `median_library_ready()` / `Median.onReady()` as this skill documents)*

### In-app purchases

The IAP plugin provides full StoreKit (Apple) and Google Play Billing support — consumables, non-consumables, subscriptions, one-time purchases. Configure the plugin with a `productsUrl` pointing at a JSON file on your site that lists product IDs (Apple: flat `products` array; Google: `inappProducts` + `subProducts` sections) — add or remove products without a new app release; Apple optionally adds a `postUrl` for server-side verification.

```javascript
// Products + status — the app calls median_info_ready(productsData) on page load,
// or fetch at runtime (Apple adds canMakePurchases; Google adds libraryVersion):
var productsData = await median.iap.info();

// Purchase — Google "subs" also need offerToken from productsData:
try {
  var purchasesData = await median.iap.purchase({ productID: "product_id" });
} catch (error) { /* error provided by Apple/Google */ }

// Purchase history — the app calls median_iap_purchases(purchasesData) on load
// and after purchases, or fetch at runtime:
var purchasesData = await median.iap.purchases();

median.iap.restorePurchases();                              // Apple: restore previous purchases
median.iap.manageSubscription({ productID: "product_id" }); // Google: manage this app's subscription
median.iap.manageAllSubscriptions();                        // Google: all of the user's subscriptions
```

Verification: on Apple, server-side via the App Store Server API (send `transactionId` to your backend — recommended when purchases tie to user accounts), the deprecated `verifyReceipt` POST flow, or on-device verification (omit `postUrl` and read `hasValidReceipt`). On Google, read the purchase data (each entry carries `purchaseToken`, `purchaseState`, `acknowledged`, `autoRenewing`) and use Real-Time Developer Notifications for backend status. Google subscription upgrades/downgrades pass `previousPurchaseToken` plus a proration mode — the docs name it `replacementMode` in the parameter list but `prorationMode` in the example; `previousProductID` is no longer accepted.

Google apps built before Billing Library 6 run in `legacyMode` (Billing Library 4): check `libraryVersion` from `median.iap.info()` (6+ = new API; lower/missing = old), support both on the website during rollout, then rebuild with `legacyMode` disabled.

Apple mandates IAP for digital goods (consumables, non-consumables, non-renewing and auto-renewing subscriptions) inside iOS apps; follow both stores' payment guidelines (see https://docs.median.co/docs/faq-publishing.md). Full parameter tables, verbatim response payloads, verification flows, and testing checklists are in `references/iap.md`; deep detail stays on the official subpages:
- Apple: https://docs.median.co/docs/apple-iap.md
- Google: https://docs.median.co/docs/google-iap.md
- Overview: https://docs.median.co/docs/iap.md

### RevenueCat (managed IAP + paywall modals)

The RevenueCat plugin (released Dec 2025) is an alternative to wiring StoreKit/Play Billing yourself — it wraps both stores behind RevenueCat's backend and SDKs, "reducing the complexity of implementing Apple's StoreKit and Google Play Billing". The bridge API mirrors the IAP flow — `median.revenueCat.configure({ apiKey, appUserID })`, `isInitialized()`, `getOfferings()`, `purchase({ identifier })`, `restorePurchases()` — and adds paywall modals designed in the RevenueCat dashboard and presented natively with `median.revenueCat.presentPaywall()` (returns an error if the user cancels). Paywalls draw close App Review scrutiny — follow RevenueCat's paywall-approval guidance. Docs: https://docs.median.co/docs/revenue-cat.md • Demo: https://median.dev/revenue-cat

### Deep linking

- **Android App Links**: host `/.well-known/assetlinks.json` (proves domain control) BEFORE publishing; verify each hostname with Google's Statement List Generator:

  ```json
  [
    {
      "relation": ["delegate_permission/common.handle_all_urls"],
      "target": {
        "namespace": "android_app",
        "package_name": "co.median.android.padzoa",
        "sha256_cert_fingerprints": [
          "ED:30:0F:A9:AB:9D:00:34:9D:48:B0:91:69:83:D7:C9:FE:0A:95:FE:F2:E0:38:25:C9:97:37:D8:F3:16:0B:E0"
        ]
      }
    }
  ]
  ```

- **iOS Universal Links**: host AASA at `/apple-app-site-association` or `/.well-known/apple-app-site-association`; explicit app ID + Associated Domains in the provisioning profile; the `appID` in the file uses TEAMID:

  ```json
  {
    "applinks": {
      "apps": [],
      "details": [ { "appID": "TEAMID.co.median.example", "paths": ["*"] } ]
    }
  }
  ```

- **Custom URL schemes** (advanced): format `youruniquestring://`; the app recognizes URLs like `youruniquestring.https://example.com/path`. Lowercase only — "Uppercase letters and numbers may lead to inconsistent behavior across platforms." Not supported in desktop browsers. Commonly used for mobile auth redirects (e.g. the Auth0 callback scheme).
- Requirements/best practice: serve both files with `Content-Type: application/json`, https + valid cert, HTTP 200, no redirects; deep links need a PATH (`http://example.com` won't work, `http://example.com/PATH` will); every subdomain needs its own entry and its own file; add both root domain and `www` host. Validator: https://docs.median.co/docs/deep-linking-validator.md — Demo: https://median.dev/deep-linking/

### Offline download manager

```javascript
median.downloads.init({ callback: downloadCallback }); // returns promise — register before any downloads
median.downloads.downloadFile({ url: "URL", title: "Title" });
median.downloads.showUI();
```

`url` (required — must start with http/https), `title` (required — name shown in UI), `identifier` (optional — differentiate simultaneous downloads in the callback), `details` (optional description), `date` (optional yyyy-mm-dd, e.g. podcast publish date). The callback receives `{ identifier, event }` where event is `"progress"` (plus `bytesWritten`, `expectedBytes`), `"done"`, or `"error"` (plus `errorMessage`). Configure an offline.html page so the download manager UI can open when the app is offline for extended periods (see Offline Page docs + Codepen PoJbEEN sample). Demo: https://median.dev/offline-download-manager/

## Pitfalls

- **`intialize` is not a typo** — the Adjust docs verbatim spell it `median.adjust.intialize` (missing 'i'). Using `initialize` will fail; don't auto-correct it.
- **Firebase `collection({enabled:false})` is persistent** — it disables ALL logging including automatic events and stays off until re-enabled with `true`. Default state is logging enabled.
- **Firebase `setUser` only after authentication**, then fire a follow-up event so identity/properties attach.
- **AppsFlyer cache reads on cold launch** — `getConversionData()`/`getDeepLinkResult()` never wait for the SDK and can resolve `data: null`; prefer the global callbacks or NPM listeners for first-launch data.
- **AppsFlyer platform divergence** — deep-link payload shape differs iOS vs Android (root-level vs nested under `data`; empty object `{}` on iOS vs an `error` object on Android when absent). Read `deeplinkValue` defensively.
- **AppsFlyer plugin version gate** — conversion/deep-link APIs need plugin 1.4.0 (iOS) / 1.3.0 (Android) and are NOT yet in the NPM package.
- **IAP deep detail lives on the subpages** — `references/iap.md` carries the full tables here, but verify against the `apple-iap`/`google-iap` doc pages before shipping store flows. Apple mandates IAP for digital goods in iOS apps.
- **Bridge functions undefined** — the bridge library initializes after page load; run bridge code from `median_library_ready()` (with the `if (window.median)` manual-call fallback) or `Median.onReady()` (NPM), and confirm the plugin is enabled AND the app rebuilt after config changes.
- **Crashlytics `toastErrorLogsEnabled` is Android-only** — `webErrorLogsEnabled` (WebView console errors) works on both platforms; toast errors (e.g. SSL certificate warnings) only on Android builds.
- **Crashlytics user ID persists** — `setUserId` survives app launches and is cleared only on uninstall/reinstall; call `unsetUserId()` on logout for shared devices.
- **Google `previousProductID` is dead** — subscription upgrades/downgrades must pass `previousPurchaseToken` (docs still describe `previousProductID` but note it is "no longer accepted for purchase"); if it is empty, the proration mode is ignored.
- **Google proration naming is inconsistent upstream** — the docs document `replacementMode` values (`CHARGE_FULL_PRICE`, `DEFERRED`, ...) but the example passes `prorationMode: "IMMEDIATE_AND_CHARGE_PRORATED_PRICE"`; both spellings appear in the docs — verify on-device before relying on either.
- **Google Billing 4→6 split** — old builds run `legacyMode` (Billing 4), new builds use Billing 6; branch on `libraryVersion` from `median.iap.info()` so one website serves both during rollout.
- **IAP testing needs real stores** — Apple: sandbox users on a physical device (simulators need advanced Xcode setup), accelerated renewals (5 minutes per month, 6 times). Google: device with Google Play signed in — Appetize browser simulators do NOT support Google Play.
- **appConfig.json is crash-loaded** — invalid JSON can crash the app AT LAUNCH. "Use a JSON validator or linter before saving changes." To duplicate an app use the "Clone" top-menu feature rather than config import.
- **Plugin config changes need a rebuild** — settings changed in appConfig.json/App Studio (tokens, keys, sounds, Google Services files) require a save + REBUILD of the app to take effect.
- **Plugin tier licensing** — "Native Plugins are available based on the tier of your license. Some plugins are only available with a Business or Enterprise plan." Essential/Plus plugins are trialable (Trial Active label); unlicensed trial apps show a 'This app was developed using Median' popup until licensed. Some plugins require active third-party subscriptions.
- **Adjust environment** — set Environment to **Production** before publishing to the stores; Development is for testing only.
- **Deep-link serving rules** — https with valid cert, HTTP 200, no redirects, `Content-Type: application/json`; URL schemes lowercase only; iOS links must include a PATH.

## Verification

1. Bridge readiness: calls made inside `median_library_ready()` (or `Median.onReady()` with the NPM package) execute in-app; the same calls at raw page top-level may not.
2. Adjust: `median.adjust.trackEvent(new AdjustEvent("<token>"))` on a real device → the event (with revenue + 3-letter currency) appears in the Adjust dashboard.
3. AppsFlyer: on a fresh install, a conversion listener (`median_appsflyer_cd` or the NPM `conversionData.addListener`) receives non-null data after SDK start.
4. Firebase Analytics: `logEvent`/`logScreen` calls appear in Firebase DebugView with the consent defaults configured.
5. Crashlytics: open https://median.dev/crashlytics in-app → logs/events arrive in the Firebase Crashlytics console; `enable(false)` stops collection and `enable(true)` resumes it.
6. Crashlytics identity: after `setUserId("user_123")`, records in Firebase carry the user ID; after `unsetUserId()` they do not. Toast error logs change behavior on Android builds only.
7. IAP products: on launch `median_info_ready(productsData)` fires (or `await median.iap.info()` resolves) listing every product ID from the hosted `productsUrl` JSON with localized prices.
8. IAP purchase (sandbox tester): `median.iap.purchase({ productID })` opens the native payment sheet and resolves with `purchasesData`; `median.iap.purchases()` then lists it — Apple in `allPurchases` (+ `activeSubscriptions` for subscriptions), Google with `purchaseState: 0` and `acknowledged: true`.
9. IAP verification: with `postUrl` set (Apple), the receipt POST reaches your backend and its `{success, title, message}` response is shown in-app; with on-device mode, `median_iap_purchases` reports `hasValidReceipt: true`.
10. IAP restore/manage: Apple `restorePurchases()` re-delivers a previous non-consumable/subscription on a second device; Google `manageSubscription({ productID })` / `manageAllSubscriptions()` open the Play Store subscription screens.
11. Google Billing migration: a `legacyMode`-disabled build reports `libraryVersion` ≥ 6 from `median.iap.info()`, and the website still handles builds reporting lower/missing values.
12. Deep linking: the Deep Linking Validator (https://docs.median.co/docs/deep-linking-validator.md) passes for every hostname; tapping an https link containing a PATH opens the app, not the browser.
13. Downloads: `downloadFile({ url, title })` drives the callback through `progress` → `done`, and the file opens from the download manager UI with the device offline.

## References

- [references/analytics.md](references/analytics.md) — Adjust/AppsFlyer/Firebase Analytics/Firebase Crashlytics full parameter tables, verbatim response payloads, global callbacks, NPM listener API, plugin setup details.
- [references/iap.md](references/iap.md) — Apple/Google IAP full parameter tables, verbatim response payloads, verification flows (App Store Server API, deprecated verifyReceipt POST, on-device), restore + subscription management, Google Billing 4→6 `legacyMode` migration, testing checklists.
- [references/config-and-links.md](references/config-and-links.md) — deep-linking file examples and rules, offline download manager API + callback shape, appConfig.json rules, plugin licensing/trial behavior.
- [Repo recipes](../../recipes.md) — cross-skill combos featuring this skill
- Official docs: https://docs.median.co/docs/analytics • https://docs.median.co/docs/adjust • https://docs.median.co/docs/appsflyer • https://docs.median.co/docs/firebase-analytics • https://docs.median.co/docs/crashlytics • https://docs.median.co/docs/iap • https://docs.median.co/docs/apple-iap • https://docs.median.co/docs/google-iap • https://docs.median.co/docs/google-iap-legacy • https://docs.median.co/docs/revenue-cat • https://docs.median.co/docs/deep-linking • https://docs.median.co/docs/offline-download-manager • https://docs.median.co/docs/native-plugins-overview • https://docs.median.co/docs/app-configuration-appconfigjson
- Live demo pages: https://median.dev/iap • https://median.dev/crashlytics • https://median.dev/revenue-cat • https://median.dev/appsflyer/ • https://median.dev/firebase-analytics/ • https://median.dev/deep-linking/ • https://median.dev/offline-download-manager/
