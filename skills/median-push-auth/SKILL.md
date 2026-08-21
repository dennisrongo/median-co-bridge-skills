---
name: median-push-auth
description: "Add push notifications and native auth to a Median app."
version: 0.1.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
---

# Median Push Notifications & Native Auth

Wire OneSignal push notifications and native authentication into an existing web app running inside a Median.co app. The JS Bridge is invoked via the global `median` object, and calls should be made after the app/bridge is ready (e.g. after `deviceready`). OneSignal push uses the SDK v5+ user-centric model by default; native auth (Clerk, Auth0, social login, biometrics) runs through dedicated `median.*` namespaces.

## When to Use

Use this skill when the task involves:
- Enabling OneSignal push and controlling when the native permission prompt appears
- GDPR-style privacy consent gating (preventing OneSignal from transmitting any data)
- Associating your users with push identities (`login` / `logout` / external IDs), data tags, and badge counts
- Handling notification taps in JS (`median_onesignal_push_opened`) and foreground display
- Sending server-side push via the OneSignal REST API with `include_aliases.external_id` targeting
- Native authentication: Clerk, Auth0, Facebook/Google/Apple social login, Face ID/Touch ID/Android biometric, passkeys

Don't use for:
- Analytics or attribution (Adjust, AppsFlyer, Firebase Analytics) → use the `median-analytics-iap` skill
- In-app purchases, deep-link domain hosting (AASA/assetlinks), offline downloads, appConfig.json → use the `median-analytics-iap` skill
- Push providers other than OneSignal (Braze, Klaviyo, etc.) — OneSignal is Median's default provider and the only one documented here

## Prerequisites

- **Median app** with the relevant plugins enabled in App Studio → Native Plugins (each requires save + rebuild to take effect):
  - **OneSignal** plugin + your OneSignal App ID. Push credentials uploaded to OneSignal: APNs `.p8` token key (recommended) or `.p12` under Settings > Push & In-App > Apple iOS (APNs); Firebase project + Service Account JSON under Settings > Push & In-App > Google Android (FCM).
  - **Clerk** plugin + Clerk publishable key (`pk_test_`/`pk_live_`) (Advanced Mode).
  - **Auth0** plugin + domain/clientId/scheme/audience JSON (Advanced Mode).
  - **Social Login** plugins with provider credentials: Facebook (App ID, Display Name, Client Token), Google (iOS + Android Client IDs), Apple (iOS Bundle ID).
- **OneSignal REST API Key** — server-side secret only; never ship it in client JS or app code.
- **iOS capabilities** (source builds / manual signing): Push Notifications + Background Modes (Remote notifications); for badge counts, rich images, and confirmed delivery also App Groups (`group.YOUR_BUNDLE_IDENTIFIER.onesignal`) on both the main app target and the OneSignalNotificationServiceExtension target.
- **Physical device for testing** — push and biometrics don't work reliably on simulators. Android ≤12 grants push permission at install time (no prompt).

## Quick Reference

| Task | Call |
| --- | --- |
| Prompt for push permission now | `median.onesignal.register()` |
| Grant / revoke GDPR privacy consent | `median.onesignal.userPrivacyConsent.grant()` / `median.onesignal.userPrivacyConsent.revoke()` |
| Associate your user with OneSignal | `median.onesignal.login("user@domain.com")` |
| Disassociate on app logout | `median.onesignal.logout()` |
| Get user + subscription info | `await median.onesignal.info()` (or `median.onesignal.onesignalInfo()`) |
| Set / get / delete data tags | `median.onesignal.tags.setTags({ tags: {...} })` / `getTags()` / `deleteTags({ tags: [...] })` |
| Show native tags UI | `median.onesignal.showTagsUI()` |
| Set / clear badge count | `median.onesignal.badgeCount.set(5)` / `median.onesignal.badgeCount.set(0)` |
| Show notifications while app is open | `median.onesignal.enableForegroundNotifications(true)` |
| React to a notification tap | define global `function median_onesignal_push_opened(data)` |
| Send push from your backend | `POST https://api.onesignal.com/notifications` |
| Clerk native sign-in UI | `await median.clerk.presentSignIn()` |
| Clerk init (required on iOS) | `median.clerk.initialize({ publishableKey: 'pk_...' })` |
| Clerk status + fresh token | `await median.clerk.getAuthStatus()` |
| Auth0 Universal Login | `await median.auth0.login({ scope: "email profile offline_access" })` |
| Auth0 stored credentials (auto-renews) | `await median.auth0.getCredentials()` |
| Auth0 renew manually | `median.auth0.renew({ refreshToken? })` |
| Facebook / Google / Apple login | `median.socialLogin.facebook.login({...})` / `.google.login({...})` / `.apple.login({...})` |
| Prompt biometric auth | `median.auth.authenticate()` |
| Passkeys | pure WebAuthn in your web code — no bridge calls |

## How It Works

### Push permission strategy

Four levels of control, from loosest to strictest:

1. **Default** — plugin enabled, OneSignal initializes on launch and prompts for push permission on first open.
2. **Delayed registration** — App Studio → Native Plugins > OneSignal → disable "Auto-register for push notifications", then prompt when the user is ready:

   ```html
   <a onclick="median.onesignal.register()">Enable push notifications</a>
   ```

   Call `register()` only once per session — repeated calls after the user responded won't re-show the prompt. Prompt-delaying applies to iOS and Android 13+ only. Note: with auto-register off, OneSignal still initializes in the background and generates a `oneSignalUserId` — for a full GDPR hold, use consent instead.
3. **Privacy consent gate** — App Studio → Native Plugins > OneSignal → enable "Require user privacy consent before transmitting data". OneSignal then does not initialize at all (no `oneSignalUserId` until consent):

   ```javascript
   median.onesignal.userPrivacyConsent.grant();
   median.onesignal.userPrivacyConsent.revoke();
   ```

   Revoking stops data transmission but does NOT stop push delivery to an already-opted-in device — use Data Tags or `median.onesignal.logout()` for that.
4. **Soft prompt** — an in-app message (OneSignal In-App Message, HTML Composer or API) shown BEFORE the native dialog to maximize opt-in. "Allow" triggers `median.onesignal.register()`; "Maybe later" closes without the native prompt. iOS shows the native dialog only once — after "Don't Allow" the user must change it in Settings.

### Identifying users (login, info, tags)

Call `login` as soon as the user authenticates (login confirmation page or right after a session check) — don't wait for another user action. External IDs are strings, max 128 characters; rejected placeholder values (e.g. `null`, `none`, `0`, `NaN`, `-`, `UNQUALIFIED`, `INVALID_USER`, and others — full list in references) fail the assignment and leave the user in its previous state.

```javascript
median.onesignal.login("user@domain.com");   // associate
median.onesignal.logout();                   // disassociate on app logout
```

Retrieve identifiers with `info()` — three retrieval methods exist (automatic `median_onesignal_info(data)` callback defined synchronously at page load, manual `median.onesignal.info({ callback: "median_onesignal_info" })`, and promise-based `await median.onesignal.info()` / `median.onesignal.onesignalInfo()` — most flexible for SPAs). Key fields: `oneSignalId` (OneSignal's user ID), `externalId` (yours, via `login()`), `subscription.id` (device-stable push subscription), `subscription.token`, `subscription.optedIn`, `requiresUserPrivacyConsent`. Full field table and the login-page POST pattern are in `references/onesignal-push.md`.

```javascript
median.onesignal.tags.setTags({ tags: { category: "sports", plan: "free", lastSeen: "2024-01-15" } })
  .then(function (tagResult) { console.log(tagResult); /* { success: true } */ });
median.onesignal.tags.getTags().then(function (r) { console.log(r); /* { success: true, tags: {...} } */ });
median.onesignal.tags.deleteTags({ tags: ["category", "lastSeen"] }).then(...); // specific keys
```

Alternative native UI: host a JSON file (example: https://median.dev/onesignal/tags.json), set "Data Tags Native UI JSON URL" in App Studio, then `median.onesignal.showTagsUI();`.

### Notification taps, badges, foreground display

Define a global handler; the app calls it automatically when a push is opened, passing the notification's Additional Data key-value pairs:

```javascript
function median_onesignal_push_opened(data) {
    console.log(JSON.stringify(data));
}
// e.g. { 'airport': 'sfo', 'direction': 'outbound', 'dateRange': 'week' }
```

Tap routing: if the payload contains `targetUrl` → navigate inside the app; custom data present → call the JS handler; neither → open the app home screen. `targetUrl` is a RESERVED key that auto-triggers in-app navigation — for conditional navigation use a different key name (e.g. `openUrl`, `deepLink`) and navigate yourself in the handler. Prefer `targetUrl` (Additional Data) over the composer's Launch URL: `targetUrl` opens INSIDE the app with full JS Bridge support; Launch URL opens a popup browser with NO JS Bridge access.

```javascript
median.onesignal.badgeCount.set(5);   // set badge
median.onesignal.badgeCount.set(0);   // clear badge (iOS all devices; Android OEM-dependent)
median.onesignal.enableForegroundNotifications(true);   // show while app open (suppressed by default)
```

`enableForegroundNotifications` works from any page, takes effect immediately for the current session, no rebuild needed; a build-time default can be set in App Studio. Badge count management is NOT available on the legacy OneSignal plugin (iOS v3 SDK / Android v4 SDK).

### Sending push from your backend (OneSignal REST API)

This is NOT a JS bridge API — send from your server with the REST API Key (`Authorization: Key ***`), never from client code:

```shell
curl --location 'https://api.onesignal.com/notifications' \
--header 'Authorization: Key ***' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data '{
  "app_id": "{{YOUR_ONESIGNAL_APP_ID}}",
  "target_channel": "push",
  "headings": { "en": "Your order has shipped", "es": "Tu pedido ha sido enviado" },
  "contents": { "en": "Order #1042 is on its way. Tap to track.", "es": "El pedido #1042 está en camino. Toca para rastrear." },
  "data": { "targetUrl": "https://yourapp.com/orders/1042" },
  "include_aliases": { "external_id": [EXTERNAL_ID] }
}'
```

`target_channel` is always `"push"`; `"en"` is required in headings/contents; `include_aliases.external_id` (recommended — maps to your user database) accepts multiple users per request; alternatives are `include_aliases.onesignal_id`, custom aliases, `included_segments`/`excluded_segments`, and `include_subscription_ids` for a single device. Custom icons/sounds (including `ios_sound`/`android_channel_id` fields) are covered in `references/onesignal-push.md`.

### Native auth overview

IETF BCP (RFC 8252) requires user login to happen in a browser session facilitated by a native app, NOT in an embedded webview — Google, Facebook, and Auth0 hosted login make this mandatory. Median's Social Login and Auth0 plugins use native SDKs and are the compliant paths. Detect Median vs browser with `navigator.userAgent.indexOf("median") >= 0` and swap browser-only/native-only buttons.

### Clerk

```javascript
const result = await median.clerk.presentSignIn();
// { state: "signedIn" | "signedOut", userId: "user_2abc...", hasValidToken: true | false, token: "eyJ..." }
median.clerk.presentSignIn({ callback: function(result) { ... } });  // callback form

const result = await median.clerk.signOut();
// { success: true | false, error?: { code: "NOT_INITIALIZED" | "SDK_ERROR", message: "..." } }

const status = await median.clerk.getAuthStatus();
if (status.state === 'signedIn' && status.hasValidToken) {
  fetch('https://api.example.com/data', { headers: { Authorization: 'Bearer ' + status.token } });
}
```

Call `getAuthStatus()` immediately before each API request for a fresh token — never cache. A dismissed sign-in flow returns `state: "signedOut"` with no token. Demo: https://median.dev/clerk/

### Auth0

Plugin config in App Studio → Native Plugins → Advanced Mode:

```json
{
  "domain": "dev-tax7avdd0xmeaavx.us.auth0.com",
  "clientId": "wTD****************KBwV",
  "scheme": "demo",
  "audience": "https://median-test.auth0.com/api/v2/"
}
```

(`scheme` matches your Android Callback URL scheme; `audience` is optional per tenant config.)

```javascript
const credentials = await median.auth0.login({ scope: "email profile offline_access" });
// { accessToken, idToken, scope, error?, refreshToken? }  // refreshToken with offline_access

median.auth0.login({ enableBiometrics: true, callback: median_auth0_post_login });  // + biometric re-auth

const auth0Status = await median.auth0.status();
// { biometryAvailable: boolean, biometryType: 'touchId' | 'faceId' | 'none', hasValidCredentials: boolean }

const credentials = await median.auth0.getCredentials();  // auto-renews using stored refreshToken
const credentials = median.auth0.renew({ refreshToken: "..." });  // optional; defaults to saved token
```

Docs quirks preserved verbatim in `references/native-auth.md`: the docs show `median.auth0.logout.then(...)` with NO parentheses (treat logout as promise-returning), and `const credentials median.auth0.renew({...})` missing its `=`. Setup requires a NEW Native Application in Auth0, callback URLs like `demo://{domain}/android/{package}/callback`, matching Deep Linking + URL Scheme config, and the iOS bundle ID / Android package in Auth0 Advanced Settings > Device Settings.

### Social login (Facebook / Google / Apple)

```javascript
median.socialLogin.facebook.login({ 'callback': <function>, 'scope': '<text>', forceLimitedLogin: true | false, nonce: '<text>' });
median.socialLogin.google.login({ 'callback': <function> });
median.socialLogin.apple.login({ 'callback': <function>, 'scope': '<text>' });
```

`callback` (required) receives the token + user details: Facebook → `{ accessToken, userId, type: "facebook", userDetails: {...}, authToken?, nonce?, limitedLogin }`; Google → `{ idToken, type: "google" }`; Apple → `{ idToken, code, firstName, lastName, type: "apple" }`. Errors come back as `{ "error": "...", "type": "<provider>" }`. Full verbatim response objects (including Facebook's `userDetails` fields) are in `references/native-auth.md`. Platform notes: Facebook iOS with ATT declined falls back to Limited Login (`authToken` JWT is NOT usable for the Graph API); Apple returns firstName/lastName only on the FIRST authentication — persist them then. Demo: https://median.dev/social-login/

### Biometrics and passkeys

- **Face ID / Touch ID / Android Biometric**: the biometric overview page routes to `apple-face-id-touch-id` and `android-biometric-auth` subpages for the full API; the call cited on the overview is `median.auth.authenticate()`. Biometrics cannot be tested on iOS/Android simulators. See https://docs.median.co/docs/auth.md and its subpages.
- **Passkeys**: "The WebAuthn / Passkey functionality operates entirely within your web implementation. It does not require the Median JavaScript Bridge." No `median.*` calls — your standard WebAuthn flow (server challenge → device signs → server verifies) is bridged to native hardware. Setup: Android hosts `/.well-known/assetlinks.json` + WebAuthn plugin `allowedUrls`; iOS needs no plugin, just a `webcredentials` entry in `/apple-app-site-association`. Details in `references/native-auth.md`.

## Pitfalls

- **Consent ordering matters**: privacy consent (`userPrivacyConsent`) is stricter than disabling auto-register — with consent required, OneSignal doesn't initialize at all and no `oneSignalUserId` is generated until `grant()`. Don't rely on "auto-register off" for a GDPR hold.
- **`register()` once per session** — iOS shows the native dialog only once; after "Don't Allow" the user must fix it in Settings.
- **`targetUrl` is reserved** — including it auto-triggers in-app navigation. Use your own key (`openUrl`, `deepLink`) if you want conditional routing in `median_onesignal_push_opened`. Never use the composer Launch URL for in-app routing (popup browser, no JS Bridge).
- **REST API Key stays server-side.** Never in client JS or app code.
- **Clerk iOS initialization contradiction (docs conflict)**: one section says the Clerk SDK auto-initializes at app startup; the troubleshooting section says on iOS it does NOT and you must call `median.clerk.initialize({ publishableKey: 'pk_...' })` before `presentSignIn`/`signOut`/`getAuthStatus` (Android: optional, initializes from server config). Treat iOS auto-init as unreliable — call `initialize()` on iOS and check for `success: true`.
- **Auth0 doc typos (flagged as-is, don't copy blindly)**: `median.auth0.logout.then(...)` is shown without parentheses; `const credentials median.auth0.renew({...})` is shown missing `=`. Both are treated as promise-returning calls.
- **External ID validation**: rejected placeholder values (full list in references) and >128 chars fail the assignment silently-ish — the user stays in their previous state.
- **Apple social login**: firstName/lastName only on FIRST authentication — persist immediately; later logins return only the JWT.
- **Facebook Limited Login** (ATT declined): you get `authToken` (JWT) + `nonce`, NOT an `accessToken` — validate per Facebook's Limited Login token docs.
- **Rebuild requirements**: plugin settings changed in App Studio/appConfig.json (App ID, auto-register, consent requirement, sounds, icons) need a save + REBUILD. Runtime exceptions: `enableForegroundNotifications()` (immediate, current session) and the OneSignal REST API.
- **Legacy mode**: apps built before OneSignal SDK v5 use the device-centric Legacy Mode; badge count management is NOT available there (iOS v3 / Android v4 SDK).
- **Simulators**: push and biometrics don't work on most simulators — test on physical devices. Confirmed delivery analytics additionally require an eligible (paid) OneSignal plan even with App Groups configured.

## References

- `references/onesignal-push.md` — full OneSignal API: setup/credentials, register + consent flow, `oneSignalInfo` field table, all three retrieval methods, tags API, REST targeting table + payloads, badge counts, icons/sounds tables, tap handler routing.
- `references/native-auth.md` — Clerk/Auth0/social login full parameter tables and verbatim response objects, error codes, passkey AASA/assetlinks setup, biometric subpage links.
- Official docs: https://docs.median.co/docs/onesignal.md • https://docs.median.co/docs/user-consent-management.md • https://docs.median.co/docs/identifying-targeting-users.md • https://docs.median.co/docs/programmatic-notifications.md • https://docs.median.co/docs/handling-notifications-taps.md • https://docs.median.co/docs/notification-customization.md • https://docs.median.co/docs/foreground-notifications.md • https://docs.median.co/docs/authentication.md • https://docs.median.co/docs/clerk.md • https://docs.median.co/docs/auth0.md • https://docs.median.co/docs/social-login.md • https://docs.median.co/docs/passkey-authentication.md
