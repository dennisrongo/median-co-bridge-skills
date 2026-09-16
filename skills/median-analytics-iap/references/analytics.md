# Analytics & Attribution — Full API Reference (Median JS Bridge)

All call signatures, parameters, and JSON shapes verbatim from docs.median.co as of 2026-08-20 (Firebase Crashlytics and In-App Purchases sections verified 2026-09-12). Doc quirks (e.g. the Adjust `intialize` spelling) are preserved verbatim and flagged.

## Adjust

Purpose: Adjust SDK init, event tracking (incl. revenue), and attribution retrieval.

### Call signatures (verbatim)
```javascript
// Manual init; optional boolean enables/disables SKAN (iOS only, ignored on Android)
median.adjust.intialize(true | false);

const adjustEvent = new AdjustEvent("abc123");   // event token from Adjust dashboard
adjustEvent.setRevenue(0.01, "EUR");
median.adjust.trackEvent(adjustEvent);

median.adjust.attributionInfo();
```

NOTE ON SPELLING: the docs consistently spell the init function `intialize` (missing 'i') — verbatim, not a transcription error. Use `median.adjust.intialize(...)`.

### Parameters (verbatim)

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| SKAN enabled flag (intialize) | `boolean` | No | When Manual initialization is selected, enables or disables StoreKit Ad Network Attribution on iOS for that session. |
| `adjustEvent` (trackEvent) | `AdjustEvent` | Yes | Instance created with your Adjust event token; optional revenue via `setRevenue(amount, currency)`. |

Return: `attributionInfo()` returns attribution information stored by Adjust for the current user/install.

### App Studio settings
- App Token (from Adjust dashboard AppView → app → App information → App details)
- Environment (**Production**/Development — set Production before publishing)
- Initialization (**Automatic**/Manual)
- StoreKit Ad Network Attribution (Enable/Disable, iOS)

### Gotchas
- Only call `intialize` when Initialization=Manual.
- Event tokens are created in the Adjust dashboard, not Median.
- `setRevenue(amount, currency)` needs a 3-letter currency code.
- Organic installs may return organic/limited attribution data.
- Test on real devices.

## AppsFlyer

Purpose: user association, custom event logging, conversion/deep-link data.

### Call signatures (verbatim, promise + NPM forms)
```javascript
const { success } = await median.appsflyer.setCustomerUserId("user123");
const { success } = await median.appsflyer.logEvent("purchase", { value: 99.99, currency: "USD" });

const conversion = await median.appsflyer.getConversionData();
const deepLink = await median.appsflyer.getDeepLinkResult();
```
NPM package: `import Median from "median-js-bridge";` then `Median.appsflyer.setCustomerUserId(...)`, `Median.appsflyer.logEvent(...)`.

### Parameters (verbatim)

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `userId` | `string` | Yes | Your unique customer or user |
| `eventName` | `string` | Yes | Event name (e.g. `'purchase'`, `'signup'`) |
| `eventValues` | `object` | No | Key-value pairs to attach to the event |

### Response shapes (verbatim)
```json
// getConversionData / cached read
{ "success": true, "data": { "af_status": "Organic", "is_first_launch": false } }
// nothing cached / no deep link
{ "success": false, "data": null }
// conversion callback payload
{ "is_first_launch": false, "install_time": "2026-02-11 10:32:16.951",
  "af_message": "organic install", "af_status": "Organic" }
// deep link found (iOS root-level; Android 1.4.0+ nests under data)
{ "afSub1": "", "afSub2": "", "afSub3": "", "afSub4": "", "afSub5": "",
  "campaign": "summer_sale", "campaignId": "1234567890", "clickEvent": {},
  "clickHTTPReferrer": "", "deeplinkValue": "product/42", "isDeferred": true,
  "matchType": "referrer", "mediaSource": "email" }
// no deep link: Android { "success": false, "data": null, "error": { "message": "not_found" } } ; iOS {}
```

### Global callback functions (verbatim)
```javascript
function median_appsflyer_cd_success(conversionDataMap) { /* legacy install/conversion */ }
function median_appsflyer_cd(data) { /* new conversion data */ }
function median_appsflyer_deeplink_result(data) { /* UDL deep link; data.deeplinkValue */ }
```

### NPM listeners
`Median.appsflyer.conversionData.addListener(fn)`, `Median.appsflyer.deeplinkResult.addListener(fn)`, `Median.appsflyer.sdkStart.addListener(fn)` — each returns an ID for `removeListener(id)`.

### Platform notes & gotchas
- `af_status` is "Organic" or "Non-organic".
- iOS 14.5+ requires ATT (configurable ATT prompt delay in the plugin).
- `getConversionData`/`getDeepLinkResult` are cache reads — "they never wait for the SDK, so on a cold launch they can resolve with `data: null`".
- Requires AppsFlyer plugin 1.4.0 (iOS) / 1.3.0 (Android); NOT yet in the NPM package.
- Unset campaign fields come back as empty strings, not omitted.
- Deep-link payload shape differs by platform — "Read `deeplinkValue` defensively rather than branching on the shape."
- Config: Android Dev Key, Apple Dev Key, Apple App ID in Native Plugins > AppsFlyer > Settings.
- Demo: https://median.dev/appsflyer/

## Firebase Analytics

Purpose: Firebase Analytics runtime control — consent, collection toggle, user identity, events, screens, eCommerce.

### Setup
Upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) in App Studio Build & Deploy > Google Services; set default consent (Follow EU Consent Policy / Deny / Allow); enable the plugin.

### Call signatures + parameters (verbatim)
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

### Parameter tables (verbatim)

| Call | Param | Type | Required | Notes |
| --- | --- | --- | --- | --- |
| setConsent | Consent object | `object` | Yes | Boolean keys: adStorage, analyticsStorage, adUserData, adPersonalization |
| collection | `enabled` | `boolean` | Yes | Wrapped in `{ enabled: ... }`; persistent — re-enable with `true` |
| setUser | `ID` | `string` | Yes | Passed as `'ID'` on the object |
| setUserProperty | `key`/`value` | `string` | Yes | Property name and value |
| defaultEventParameters | `data` | `object` | Yes | Key-value pairs |
| logEvent | `event` | `string` | Yes | Event name |
| logEvent | `data` | `object` | No | Arbitrary event payload |
| logScreen | `screen` | `string` | Yes | Screen name |

### ProductItem optional fields
All optional; recommend at least `item_id` or `item_name`:
```javascript
String item_id; String item_name; String item_category; String item_variant;
String item_brand; String item_list_name; String item_list_id; double price;
```

### Gotchas
- Logging enabled by default; `collection({enabled:false})` disables ALL logging incl. automatic events and persists until re-enabled.
- setUser only after authentication; fire a follow-up event so identity/properties attach.
- Firebase auto-logs screen_view, session_start, first_open_time by default.
- Demo: https://median.dev/firebase-analytics/

## Firebase Crashlytics

Purpose: real-time crash reporting — native crashes, WebView console errors, and (Android only) toast errors, with runtime control via the bridge.

### Setup
Same gate as Firebase Analytics: register the app in a Firebase project with Crashlytics enabled; upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) in App Studio under Build & Deploy > Google Services; enable the plugin in Native Plugins; save + rebuild.

### Call signatures (verbatim)
```javascript
// Enable/disable Crashlytics
median.firebaseCrashlytics.enable(true);
median.firebaseCrashlytics.enable(false);

// WebView error logging
median.firebaseCrashlytics.webErrorLogsEnabled(true);
median.firebaseCrashlytics.webErrorLogsEnabled(false);

// Toast error logging (Android only)
median.firebaseCrashlytics.toastErrorLogsEnabled(true);
median.firebaseCrashlytics.toastErrorLogsEnabled(false);

median.firebaseCrashlytics.setUserId("user_123");
median.firebaseCrashlytics.unsetUserId();
```

### Parameters (verbatim)

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `enabled` (enable / webErrorLogsEnabled / toastErrorLogsEnabled) | `boolean` | Yes | `true` to enable, `false` to disable |
| User ID (setUserId) | `string` | Yes | Your user identifier — tracks crash logs per user in Firebase |

### Plugin settings
- `requestOptIn` — enable/disable the consent prompt for log collection; when true, users see a permission dialog on first app launch after installation
- `toastErrorLogsEnabled` — Android-only toast error collection (e.g. SSL certificate warnings)
- `webErrorLogsEnabled` — WebView console error collection

### Gotchas
- `toastErrorLogsEnabled` applies to Android only — test on an Android build.
- `setUserId` persists between app launches; cleared only on uninstall/reinstall.
- Functions undefined or not working → plugin enabled, app rebuilt, Google Services files uploaded, and calls made after the bridge library is ready (`median_library_ready()` / `Median.onReady()`).
- Demo: https://median.dev/crashlytics

## In-App Purchases

Full StoreKit (Apple) and Google Play Billing support: consumables, non-consumables, subscriptions, one-time purchases. The bridge API — `median.iap.info()` / `median_info_ready`, `median.iap.purchase({ productID, offerToken, previousPurchaseToken, prorationMode/replacementMode })`, `median.iap.purchases()` / `median_iap_purchases`, `median.iap.restorePurchases()` (Apple), `median.iap.manageSubscription({ productID })` / `median.iap.manageAllSubscriptions()` (Google), verification flows, and the Google Billing 4→6 `legacyMode` migration — is documented with full parameter tables and verbatim payloads in `iap.md` (this folder).

Deep detail remains on the official subpages:
- Apple IAP: https://docs.median.co/docs/apple-iap.md
- Google IAP: https://docs.median.co/docs/google-iap.md
- Overview: https://docs.median.co/docs/iap.md

Apple mandates IAP for digital goods (consumables, non-consumables, non-renewing and auto-renewing subscriptions) inside iOS apps; follow both stores' payment guidelines (see https://docs.median.co/docs/faq-publishing.md).

## Sources

- https://docs.median.co/docs/analytics.md
- https://docs.median.co/docs/adjust.md
- https://docs.median.co/docs/appsflyer.md
- https://docs.median.co/docs/firebase-analytics.md
- https://docs.median.co/docs/crashlytics.md
- https://docs.median.co/docs/iap.md
- https://docs.median.co/docs/apple-iap.md
- https://docs.median.co/docs/google-iap.md
