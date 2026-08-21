# Config, Deep Linking, Downloads & Licensing — Full Reference (Median JS Bridge)

All examples and rules verbatim from docs.median.co as of 2026-08-20.

## Deep Linking (Universal Links / App Links / URL schemes)

Purpose: let http(s) links open in the app instead of the browser; custom URL schemes for auth redirects.

### Android App Links
Host `/.well-known/assetlinks.json` (proves domain control) BEFORE publishing; verify each hostname with Google's Statement List Generator. Example (verbatim):
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

### iOS Universal Links
Host AASA at `/apple-app-site-association` or `/.well-known/apple-app-site-association`; explicit app ID + Associated Domains in provisioning profile; appID in the file uses TEAMID. Example (verbatim):
```json
{
  "applinks": {
    "apps": [],
    "details": [ { "appID": "TEAMID.co.median.example", "paths": ["*"] } ]
  }
}
```

### iOS gotchas (verbatim)
- Deep links require a PATH (`http://example.com` won't work, `http://example.com/PATH` will).
- Every subdomain needs its own Associated Domains entry AND its own AASA file.
- Serve with `Content-Type: application/json`, https + valid cert, HTTP 200, no redirects.

### Custom URL schemes (advanced)
- Format `youruniquestring://`; the app recognizes URLs like `youruniquestring.https://example.com/path`.
- Lowercase only — "Uppercase letters and numbers may lead to inconsistent behavior across platforms."
- Not supported in desktop browsers.
- Commonly used for mobile auth redirects (e.g. Auth0 callback scheme).

### Best practice
- Add both root domain and `www` host.
- Validator: https://docs.median.co/docs/deep-linking-validator.md
- Demo: https://median.dev/deep-linking/

## Offline Download Manager

Purpose: download documents/media for offline access; built-in file manager UI + programmatic control.

### Call signatures (verbatim)
```javascript
median.downloads.init({ callback: downloadCallback }); // returns promise — register before any downloads
median.downloads.downloadFile({ url: "URL", title: "Title" });
median.downloads.showUI();
```

### Parameters (verbatim)
- `url` (required — must start with http/https)
- `title` (required — name shown in UI)
- `identifier` (optional — string to differentiate simultaneous downloads in the callback)
- `details` (optional — description shown below title)
- `date` (optional — yyyy-mm-dd shown below details, e.g. podcast publish date)

### Callback shape (verbatim)
Callback receives `{ identifier, event }` where event is `"progress"`, `"done"`, or `"error"`:
- progress: also `bytesWritten` (bytes downloaded), `expectedBytes` (server-indicated size)
- error: also `errorMessage` (string reason)

### Gotchas
- Configure an offline.html page in the app so the download manager UI can open when the app is offline for extended periods (see Offline Page docs + Codepen PoJbEEN sample).
- Demo: https://median.dev/offline-download-manager/

## appConfig.json

Purpose: the core key-value datastore for the app — branding assets, interface settings, native plugin configurations.

- Editing: App Studio UI or direct source edit.
- Import full config (incl. assets) from another app: App Studio > Build & Deploy > App Configuration > Import from existing app.

### Gotchas
- Invalid JSON in appConfig.json can crash the app AT LAUNCH — "Use a JSON validator or linter before saving changes."
- To duplicate an app, use the "Clone" top-menu feature rather than config import (safer).
- Plugin settings changed in appConfig.json/App Studio require a save + REBUILD of the app to take effect (per the OneSignal and Firebase pages).

## Native Plugins licensing & trial behavior

Key facts (verbatim where quoted):
- "Native Plugins are available based on the tier of your license. Some plugins are only available with a Business or Enterprise plan."
- Essential and Plus plugins are trialable: Native Plugins section → click + next to the plugin → "Trial Active" label if unlicensed.
- Trial apps show a 'This app was developed using Median' popup until licensed.
- Some plugins require active third-party subscriptions.
- Median supports private/custom plugins (contact sales).
- Custom plugin development can integrate any third-party SDK into the App Studio build platform.

## In-App Purchases (router pointer)

The `iap` overview page does not carry the bridge API — the implementation detail lives on:
- Apple IAP (StoreKit): https://docs.median.co/docs/apple-iap.md
- Google IAP (Play Billing): https://docs.median.co/docs/google-iap.md

Apple mandates IAP for digital goods (consumables, non-consumables, non-renewing and auto-renewing subscriptions) inside iOS apps; follow both stores' payment guidelines (https://docs.median.co/docs/faq-publishing.md).

## Sources

- https://docs.median.co/docs/deep-linking.md
- https://docs.median.co/docs/offline-download-manager.md
- https://docs.median.co/docs/app-configuration-appconfigjson.md
- https://docs.median.co/docs/native-plugins-overview.md
- https://docs.median.co/docs/iap.md
