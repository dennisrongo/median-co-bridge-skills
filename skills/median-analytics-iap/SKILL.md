---
name: median-analytics-iap
description: "Analytics, attribution, in-app purchases, deep links."
version: 0.1.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
---

# Median Analytics, Attribution, IAP & Deep Links

Add mobile analytics and attribution SDKs (Adjust, AppsFlyer, Firebase Analytics), in-app purchases, deep linking, and offline downloads to an existing web app running inside a Median.co app. The JS Bridge is invoked via the global `median` object (or `import Median from "median-js-bridge"` where the docs show the NPM form); calls should be made after the app/bridge is ready (e.g. after `deviceready`). Analytics plugins wrap native SDKs — they need App Studio setup (tokens, config files) plus a save + rebuild before the bridge calls work.

## When to Use

Use this skill when the task involves:
- Adjust event tracking and attribution (`median.adjust.*`)
- AppsFlyer user association, events, conversion data, and deep-link results (bridge + NPM listeners)
- Firebase Analytics consent, collection toggle, user identity, events, screens, eCommerce
- Enabling in-app purchases (Apple StoreKit / Google Play Billing) — overview and where the real APIs live
- Configuring deep linking: iOS Universal Links (AASA), Android App Links (assetlinks.json), custom URL schemes
- The offline download manager (`median.downloads.*`)
- Understanding plugin tier licensing/trial behavior or appConfig.json rules that gate all of the above

Don't use for:
- Push notifications or native auth (OneSignal, Clerk, Auth0, social login, passkeys) → use the `median-push-auth` skill
- Braze, Klaviyo, Meta App Events, Crashlytics, Branch, Intent Edge — plugins exist but their APIs are outside this skill's scope (see https://docs.median.co/docs/analytics.md)

## Prerequisites

- **Median app** with the relevant plugins enabled in App Studio → Native Plugins (each change requires save + rebuild):
  - **Adjust**: App Token (Adjust dashboard AppView → app → App information → App details); Environment (**Production**/Development — set Production before publishing); Initialization (Automatic/Manual); StoreKit Ad Network Attribution (iOS).
  - **AppsFlyer**: Android Dev Key, Apple Dev Key, Apple App ID in Native Plugins > AppsFlyer > Settings. Requires AppsFlyer plugin 1.4.0 (iOS) / 1.3.0 (Android) for the conversion/deep-link APIs; they are NOT yet in the NPM package.
  - **Firebase Analytics**: upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) under Build & Deploy > Google Services; set default consent (Follow EU Consent Policy / Deny / Allow); enable the plugin.
  - **In-App Purchases**: products created in App Store Connect / Google Play Console; see the apple-iap and google-iap doc pages (referenced below) for the bridge API.
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
| In-app purchases | enable apple-iap / google-iap plugins — see official subpages for the API |
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

### In-app purchases

The IAP plugin provides full StoreKit (Apple) and Google Play Billing support — consumables, non-consumables, subscriptions, one-time purchases. The `iap` doc page is a router: the actual bridge APIs live on the `apple-iap` and `google-iap` subpages. **This skill does not replicate those APIs** — consult them directly before implementing:
- Apple: https://docs.median.co/docs/apple-iap.md
- Google: https://docs.median.co/docs/google-iap.md
- Overview: https://docs.median.co/docs/iap.md

Apple mandates IAP for digital goods (consumables, non-consumables, non-renewing and auto-renewing subscriptions) inside iOS apps; follow both stores' payment guidelines (see https://docs.median.co/docs/faq-publishing.md).

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
- **IAP router page** — don't invent bridge calls from the `iap` overview; the real APIs are on `apple-iap`/`google-iap` subpages. Apple mandates IAP for digital goods in iOS apps.
- **appConfig.json is crash-loaded** — invalid JSON can crash the app AT LAUNCH. "Use a JSON validator or linter before saving changes." To duplicate an app use the "Clone" top-menu feature rather than config import.
- **Plugin config changes need a rebuild** — settings changed in appConfig.json/App Studio (tokens, keys, sounds, Google Services files) require a save + REBUILD of the app to take effect.
- **Plugin tier licensing** — "Native Plugins are available based on the tier of your license. Some plugins are only available with a Business or Enterprise plan." Essential/Plus plugins are trialable (Trial Active label); unlicensed trial apps show a 'This app was developed using Median' popup until licensed. Some plugins require active third-party subscriptions.
- **Adjust environment** — set Environment to **Production** before publishing to the stores; Development is for testing only.
- **Deep-link serving rules** — https with valid cert, HTTP 200, no redirects, `Content-Type: application/json`; URL schemes lowercase only; iOS links must include a PATH.

## References

- `references/analytics.md` — Adjust/AppsFlyer/Firebase full parameter tables, verbatim response payloads, global callbacks, NPM listener API, plugin setup details.
- `references/config-and-links.md` — deep-linking file examples and rules, offline download manager API + callback shape, appConfig.json rules, plugin licensing/trial behavior.
- Official docs: https://docs.median.co/docs/analytics.md • https://docs.median.co/docs/adjust.md • https://docs.median.co/docs/appsflyer.md • https://docs.median.co/docs/firebase-analytics.md • https://docs.median.co/docs/iap.md • https://docs.median.co/docs/deep-linking.md • https://docs.median.co/docs/offline-download-manager.md • https://docs.median.co/docs/native-plugins-overview.md • https://docs.median.co/docs/app-configuration-appconfigjson.md
