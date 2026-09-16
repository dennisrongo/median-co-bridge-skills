# In-App Purchases — Full API Reference (Median JS Bridge)

All call signatures, parameters, and JSON shapes verbatim from docs.median.co (apple-iap page updatedAt 2026-09-09, google-iap page updatedAt 2026-09-10). Doc quirks (misspellings, example-value oddities, naming inconsistencies) are preserved verbatim and flagged in italics.

The IAP plugin provides full StoreKit (Apple) and Google Play Billing support — consumables, non-consumables, subscriptions, one-time purchases. Both stores follow the same flow: create products in the store console → host a products JSON on your site → display a web UI → initiate purchase → verify and fulfil → manage subscriptions. Developer demo: https://median.dev/iap

Apple mandates that payments for all digital goods within iOS apps be completed using their IAP platform; Google Play requires in-app purchases as an option for digital content unless that content can also be consumed outside the app (you remain free to implement other payment methods on the website itself).

## Plugin configuration (per store)

| Store | Products created in | Plugin settings (App Studio → Native Plugins → In-App Purchases) |
| --- | --- | --- |
| Apple | App Store Connect → App Store → In-App Purchases | `productsUrl` (JSON on your site; HTTP GET on launch), `postUrl` (only for server-side verification — omit for on-device verification) |
| Google | Google Play Console → Products (in-app product or subscription) | `productsUrl`; `legacyMode` (Billing Library 4 compatibility — see Migration below) |

- Apple Product IDs: alphanumeric; dots, dashes and underscores allowed; prefixing with your Bundle ID is recommended. Auto-renewing types use "subscription groups" — users can only be subscribed to one IAP in each group. At the IAP screen, click "View Shared Secret" and save the string. A user-friendly description in at least one localized language is required.
- Google: different subscription tiers (e.g. monthly and annual) go under the SAME subscription with different base plans; each base plan can have its own pricing, offers and renewal cycles.
- Hosting the JSON lets you add or remove products for sale in real time without publishing a new version of your app.

### productsUrl JSON (verbatim)

Apple:
```json
{
    "products": [
        "com.appname.subscription_monthly",
        "com.appname.subscription_annual"
        ]
}
```
*(docs prose names the second product `com.appname.subscription_yearly` while the JSON example shows `...subscription_annual` — upstream inconsistency preserved verbatim)*

Google:
```json
{
  "inappProducts": ["remove_ads"],
  "subProducts": ["subscription_monthly", "subscription_annual"]
}
```
*(docs prose says the product IDs must go in "the correct section: `inappProducts` vs `subProductions`" — `subProductions` is a docs typo; the JSON example correctly uses `subProducts`)*

## Products and status: `median.iap.info()` / `median_info_ready`

Upon launch the app verifies the product IDs with the store, then executes a JavaScript function on your page defined as `median_info_ready` with a single object parameter. You define this function, but do not actually call it — if present on a page, it is called by the app when the page is loaded. Alternatively use the bridge at runtime:

```javascript
function median_info_ready(productsData) {
   console.log(productsData);
}

// Or return the available products at runtime via a promise (in async function)
var productsData = await median.iap.info();
```

### Apple response (verbatim)

```json
{
   "inAppPurchases": {
       "platform": "iTunes",
       "canMakePurchases": true,
       "products": [{
           "productID": "product_id",
           "localizedDescription": "Description from iTunes",
           "localizedTitle": "Title from iTunes",
           "price": 9.99,
           "priceLocale": "en-US",
           "priceFormatted": "$9.99"
       }]
   }
}
```

- On iOS, `platform` is always `"iTunes"`.
- `canMakePurchases` may be `false` if disabled via parental controls or due to other reasons (see Testing).
- `products` is generated from `productsUrl`, filtered to show only purchasable items, with additional fields added.
- Display prices using `priceFormatted` — users may have different language and currency settings.

### Google response (verbatim, elisions marked)

```json
{
  "inAppPurchases": {
    "platform": "GooglePlay",
    "libraryVersion": 6,
    "products": [
      {
        "productID": "median_ad_free",
        "type": "inapp",
        "title": "Go Ad Free",
        "name": "Go Ad Free",
        "description": "Get ad-free access to the app",
        "purchaseOfferDetails": {
          "priceAmountMicros": 9000000,
          "priceCurrencyCode": "USD",
          "formattedPrice": "$9.00"
        }
      },
      {
        "productID": "median_monthly_subscription",
        "type": "subs",
        "title": "Monthly Plan",
        "name": "Monthly Plan",
        "description": "Monthly subscription for premium access",
        "subscriptionOfferDetails": [
          {
            "offerId": "60-day-trial",
            "offerToken": "AUj\/YhgwETQP...+WeW9A=",
            "basePlanId": "monthly-plan",
            "pricingPhases": [
              {
                "priceAmountMicros": 0,
                "priceCurrencyCode": "USD",
                "formattedPrice": "Free",
                "billingPeriod": "P8W4D",
                "billingCycleCount": 1,
                "recurrenceMode": 2
              },
              {
                "priceAmountMicros": 9000000,
                "priceCurrencyCode": "USD",
                "formattedPrice": "$9.00",
                "billingPeriod": "P1M",
                "billingCycleCount": 0,
                "recurrenceMode": 1
              }
            ],
            "offerTags": []
          }
          /* second offer elided: same shape, single $9.00 / P1M phase, no offerId */
        ]
      }
    ]
  }
}
```
*(the docs' own prose calls the first offer "a 30 day free trial" although its `offerId` is `60-day-trial` and its trial `billingPeriod` is `P8W4D` (8 weeks 4 days = 60 days) — upstream inconsistency preserved)*

### Google product fields (verbatim field list)

| Field | Applies to | Notes |
| --- | --- | --- |
| `productID` | all | Identifier set in Google Play; matches the `productsUrl` entries |
| `type` | all | `"inapp"` or `"subs"` |
| `title`, `name`, `description` | all | From Google Play Console |
| `purchaseOfferDetails` | `"inapp"` | `formattedPrice` (localized price to display), `priceAmountMicros` (1 USD = 1000000 micros), `priceCurrencyCode` (3-letter code) |
| `subscriptionOfferDetails` | `"subs"` | Array of offers: `offerId`, `basePlanId`, `offerTags`, `offerToken` (required to purchase that subscription offer), `pricingPhases` (time-ordered list) |
| `pricingPhases[]` | `"subs"` | `formattedPrice`, `priceAmountMicros`, `priceCurrencyCode`, `billingPeriod` (ISO 8601, e.g. `P1M`), `billingCycleCount` (number of cycles the billing period is applied), `recurrenceMode` |

`recurrenceMode` — one of 3 integer values: 1: Infinitely recurring, 2: Finitely recurring, 3: Non recurring.

*(the google-iap "Display web UI" section says "The price must be shown using the `price` string", but no `price` field exists in the product objects — the localized price field is `formattedPrice` (inside `purchaseOfferDetails`/`pricingPhases`); Apple's equivalent is `priceFormatted`. Upstream inconsistency preserved.)*

## Initiate purchase: `median.iap.purchase()`

Apple (verbatim):
```javascript
try {
  var purchasesData = await median.iap.purchase({'productID': 'product_id'});
  console.log(purchasesData);
}catch(error) {
  console.log(error);
}

//Or via a promise
median.iap.purchase({'productID': 'product_id'}).then(function(purchasesData){
  console.log(purchasesData);
}).catch(function(error){
  console.log(error);
});
```

Google (verbatim):
```javascript
var purchasesData = await median.iap.purchase({
  'productID': 'product_id', // Required
  'offerToken', 'offer_token' // Only required for "subs" type subscription purchases
});
```
*(docs show `'offerToken', 'offer_token'` — comma instead of a colon; invalid JavaScript preserved verbatim. Write `'offerToken': 'offer_token'`.)*

Your app starts the in-app purchase flow and returns `purchasesData`; if the purchase fails, an error is thrown with the error provided by Apple/Google.

### Parameters

| Parameter | Store | Required | Description |
| --- | --- | --- | --- |
| `productID` | both | Yes | Product identifier (from `productsData`) |
| `offerToken` | Google | Only for `"subs"` | Offer token from `productsData` — required to purchase that subscription offer |
| `previousProductID` | Google | No | **No longer accepted for purchase** — the docs describe it, then note it is deprecated for subscription upgrades/downgrades; use `previousPurchaseToken` instead |
| `previousPurchaseToken` | Google | For upgrades/downgrades | Replaces `previousProductID`; if this field is empty, the `prorationMode` is also ignored |
| `prorationMode` / `replacementMode` | Google | No | How to credit the user for a mid-period subscription change — the docs use BOTH names (flagged below) |

### replacementMode values (verbatim)

- `CHARGE_FULL_PRICE` — the new plan takes effect immediately, and the user is charged full price of new plan and is given a full billing cycle of subscription, plus remaining prorated time from the old plan
- `CHARGE_PRORATED_PRICE` — the new plan takes effect immediately, and the billing cycle remains the same
- `DEFERRED` — the new purchase takes effect immediately, the new plan will take effect when the old item expires *(docs wording preserved, sic)*
- `UNKNOWN_REPLACEMENT_MODE` — (no details provided)
- `WITHOUT_PRORATION` — the new plan takes effect immediately, and the new price will be charged on next recurrence time
- `WITH_TIME_PRORATION` — the new plan takes effect immediately, and the remaining time will be prorated and credited to the user

Docs upgrade example (verbatim) — note it passes `prorationMode` with a value that is not in the `replacementMode` list:
```javascript
median.iap.purchase({
  productID: "annual_membership",
  previousPurchaseToken: "aaaabbbbccccddddeeeeffff",
  prorationMode: "IMMEDIATE_AND_CHARGE_PRORATED_PRICE",
});
```
*(naming inconsistency is upstream: the bullet list documents `replacementMode` values (`CHARGE_PRORATED_PRICE`, ...) while the example uses `prorationMode: "IMMEDIATE_AND_CHARGE_PRORATED_PRICE"` — a Google Play Billing Library constant name. Both spellings appear in the docs; verify on-device before relying on either form.)*

Also from the Billing 6 migration notes: purchase history/details now contain a list of Product IDs; the single `productID` field (the first item of the list) is kept for compatibility support, so the web component receives both `productID` and `productIDs` in `median_iap_purchases` data.

## Purchase verification and fulfillment

### Apple — server-side via App Store Server API (recommended)

Recommended when purchases are associated with a user account (more secure). The `purchase` promise returns transaction info (verbatim):

```json
{
  "transactionDate": "ISOFormatedDate",
  "productId": "com.app_name.subscription_monthly",
  "transactionId": "2000123808012938"
}
```
*(docs show `"ISOFormatedDate"` — misspelling of "ISOFormattedDate" preserved verbatim)*

Send `transactionId` (and other necessary transaction parameters) to your backend via AJAX. Your server verifies with a GET to the App Store Server API Get Transaction Info endpoint `https://api.storekit.itunes.apple.com/inApps/v1/transactions/{transactionId}`, authenticated with a JWT in the authorization header (see Apple's generating-json-web-tokens-for-api-requests guide). The StoreKit API response is JWS (JSON Web Signature) encoded — decode and verify it with Apple's public keys using a well-supported library (browse https://jwt.io/libraries; confirm JWS support) rather than manually. Once decoded, treat the purchase as verified and provide access to the purchased content.

### Apple — server-side via POST call (DEPRECATED)

Uses Apple's App Store Receipts `verifyReceipt` endpoint, which Apple has deprecated in favor of the App Store Server API above. Existing apps still use this flow:

1. On purchase, the receipt data is POSTed to your configured `postUrl` with the same cookies as the logged-in user: `{ "receipt-data": "xxxxxxxxxxxxxxxxxxx" }`.
2. Your server POSTs to `https://buy.itunes.apple.com/verifyReceipt` with (verbatim):
   ```json
   {
       "receipt-data": "xxxxxxxxxxxxxxx",
       "password": "shared secret from iTunes connect",
       "exclude-old-transactions": true
   }
   ```
   (`exclude-old-transactions: true` returns only the latest transaction for auto-renewing subscriptions; otherwise you receive the entire subscription history.)
3. If Apple responds `{"status":21007}`, the receipt came from the sandbox/test environment — re-do the POST to `https://sandbox.itunes.apple.com/verifyReceipt`.
4. On status `0`: verify the receipt's `bundle_id` matches your app and which products have been purchased; save the `receipt-data` in your database as a "token" for updated subscription information (auto-renew checks); then respond (verbatim):
   ```json
   {
       "success": true,
       "title": "Thank you for your purchase!",
       "message": "Your IAP has been credited to your account",
       "loadUrl": "https://example.com/purchase-success"
   }
   ```
   `loadUrl` is optional — the app opens it after fulfilment. On any status other than 0, a non-200 response, or a failed request: do NOT fulfill; respond with `success: false` (optionally `message`, `title`, `loadUrl`). The app re-attempts the POST on every launch until it gets a success; if a purchase is never fulfilled, Apple will eventually refund the user. Log Apple's response — especially `status` — for troubleshooting.

### Apple — on-device verification

For apps without user login accounts to track across devices. Less secure (a jailbroken iPhone could theoretically make the app act as if a purchase was made). To use it, do NOT provide a `postUrl` in the app's config; the app calls `median_iap_purchases` (below) on launch and after purchases, with `hasValidReceipt` indicating validity.

### Google — purchases data + RTDN

No `postUrl`: read purchase state via `median_iap_purchases` / `median.iap.purchases()` (below) and verify the `purchaseToken` with Google server-side. For ongoing backend subscription status, configure and subscribe to a Google Cloud Pub/Sub topic for Real-Time Developer Notifications (RTDN) per the Google Play Developer documentation.

## Purchase data: `median.iap.purchases()` / `median_iap_purchases`

On app launch, and after any purchases are made, the app calls a JavaScript function on your page named `median_iap_purchases` with a single object parameter (you define it; you do not call it). Or fetch at runtime:

```javascript
function median_iap_purchases(purchasesData) {
   console.log(purchasesData);
}

// Or return the purchases at runtime via a promise (in async function)
var purchasesData = await median.iap.purchases();
```

### Apple response (verbatim)

```json
{
  "hasValidReceipt": true,
  "platform": "iTunes",
  "activeSubscriptions": ["member_basic_w"],
  "allPurchases": [
    {
      "purchaseDateString": "2019-08-11T15:53:13Z",
      "transactionIdentifier": "1000000556506948",
      "webOrderLineItemID": 1000000046196920,
      "originalPurchaseDateString": "2019-08-07T23:46:15Z",
      "quantity": 1,
      "productIdentifier": "member_basic_w",
      "originalTransactionIdentifier": "1000000555471857",
      "cancellationDateString": "",
      "subscriptionExpirationDateString": "2019-08-11T15:56:13Z"
    },
    {
      "purchaseDateString": "2019-08-11T16:03:44Z",
      "transactionIdentifier": "1000000556507336",
      "webOrderLineItemID": 1000000046196984,
      "originalPurchaseDateString": "2019-08-07T23:46:15Z",
      "quantity": 1,
      "productIdentifier": "member_basic_w",
      "originalTransactionIdentifier": "1000000555471857",
      "cancellationDateString": "",
      "subscriptionExpirationDateString": "2019-08-11T16:06:44Z"
    }
  ]
}
```

- `allPurchases` is an array of what the user's device has purchased; `activeSubscriptions` is a convenience array listing the product IDs of currently-active subscriptions.
- Typical use: gate functionality on the data (e.g. show ads unless the user has a premium ad-removal non-consumable or an active subscription).

### Google response (verbatim)

```json
{
  "platform": "GooglePlay",
  "allPurchases": [
    {
      "orderId": "GPA.3309-4129-7588-25875",
      "packageName": "co.median.android",
      "productID": "member_basic_w",
      "purchaseTime": 1567462252415,
      "purchaseState": 0,
      "purchaseToken": "fffpnbenegliokdcifadiihi.AO-J1OzGRezs5VkyKoyhYb-HgLEVG5XxswFLcLOyAnyy48sQPii2Yf6JYJe-Hm44FZT7ctkkkTlhRat15hoBWMnwXPzSgzlnCaYOvFRI_Yk5bzrBLXwOW-Mad1j9NsdoYkNywrOmJDzJ",
      "autoRenewing": true,
      "acknowledged": true,
      "purchaseTimeString": "2019-09-02T22:10:52.415Z",
      "purchaseStateString": "purchased"
    }
  ]
}
```

### Google purchase fields (verbatim field list)

- `orderId` — identifies the purchase transaction
- `packageName` — should match your app's package name
- `productID` — the identifier for what was purchased
- `purchaseTime` — milliseconds since the unix epoch (Jan 1 1970); `purchaseTimeString` — formatted as a string
- `purchaseState` — 0 – purchased, 1 – canceled, 2 – pending
- `purchaseString` — "purchased", "canceled", "pending", or "unknown" *(docs field-list name; the example JSON shows `purchaseStateString` — treat as the same field)*
- `acknowledged` — indicates the app has confirmed the purchase with Google Play
- `autoRenewing` — indicates the purchase will auto-renew; set to `false` if the user cancels the subscription
- `purchaseToken` — a string that can be used to verify the purchase with Google

The `allPurchases` array lists all current subscriptions; any expired subscriptions will no longer appear ("Any expired subscriptions will not longer appear" — docs typo preserved).

## Restoring previous purchases (Apple)

```javascript
median.iap.restorePurchases();
```

Previous purchases are restored and start the verification process as if newly purchased. If there are no purchases to restore, no action is performed — it is not possible to differentiate between a lack of purchases to restore, a delay, or a failure to restore; consider displaying a message such as "Restore requested. If you have previous purchases they will be available shortly." Offer a "Restore Purchases" button when selling subscriptions or non-consumables (users change devices or use multiple devices).

## Subscription management (Google)

```javascript
// Opens the Google Play page listing all subscriptions for all of the user's apps
// (not just your app):
median.iap.manageAllSubscriptions();

// Allows the user to manage just your app's subscription with the specified product ID:
median.iap.manageSubscription({ productID: "product_id" });
```

Apple's ongoing management is server-side: POST the receipt again to Apple's endpoint and check the `latest_receipt_info` field (optionally on a regular job across active subscriptions), and register a Status Update Notifications URL under App Store Connect → Your App → App Information.

## Upgrading the Google Play Billing Library (legacyMode)

Apps previously using the Median Google IAP plugin are configured in legacy mode to support Google Play Billing Library 4. To upgrade to Google Play Billing Library 6 you must update your website code AND submit a new build:

1. Check the `libraryVersion` in the `median_info_ready` callback or the `median.iap.info()` response — version 6 or above means the new API; a lower version number or missing value means the old API. Make your website code work with BOTH versions during the transition, while old and new app builds coexist. You can also differentiate app versions by adding custom headers or checking `appVersion` from Device Info (https://docs.median.co/docs/device-info.md).
2. Once your website supports version 6, generate a new Android build with `legacyMode` disabled: app management page → Native Plugins tab → In-App Purchases Settings → disable `legacyMode`; rebuild, complete testing, then release through Google Play.

Legacy (Billing Library 4) API reference: https://docs.median.co/docs/google-iap-legacy.md

## Testing process

### Apple

- Create App Store Sandbox users under the App Store Connect account that owns the app. Users must NOT already exist in the Apple system (no regular account logins); passwords need at least 10 characters with uppercase, lowercase, and numbers. Purchases can be tested without real payment; auto-renewing subscriptions renew at an accelerated rate (5 minutes per month) for 6 times, then cancel.
- If testing fails, especially when `canMakePurchases` is `false`, check each point: use a PHYSICAL iOS device (simulator devices require advanced Xcode setup for IAP); enable the Apple App Store In-App Purchase plugin and rebuild after saving; verify the In-App Purchase capability is in the App Studio signing setup (or Xcode Signing & Capabilities when building from source); do not use a sandbox login to sign in directly to iCloud — only use it when the purchase prompt appears; use a sandbox login created under your App Store Connect account; accept all outstanding Apple agreements for the sandbox account on iCloud.com; verify parental controls are not preventing purchases; reboot the device.

### Google

- Test devices (including simulator devices) must have Google Play Store installed and be signed into a Gmail or Google Apps for Business account. **Appetize browser-based simulators do not support Google Play** — use a physical Android device or a local simulator.
- Add tester email addresses under Developer Account → Account details → License Testing. Create an internal test track, add the test users to a user list, have each accept the Opt-in URL (`https://play.google.com/apps/internaltest/...`), and upload a release build (e.g. the Median-built apk) to the track. Testers then install the app from the Play Store; subsequent track releases are immediately available.
- In-app purchases test without actual payment; auto-renewing subscriptions renew every 5 minutes until they are canceled.

## Sources

- https://docs.median.co/docs/apple-iap.md
- https://docs.median.co/docs/google-iap.md
- https://docs.median.co/docs/iap.md
- https://docs.median.co/docs/google-iap-legacy.md
- https://docs.median.co/docs/faq-publishing.md
