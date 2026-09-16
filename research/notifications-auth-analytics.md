# Median.co JavaScript Bridge — NOTIFICATIONS / AUTH / ANALYTICS+MONETIZATION / APP CONFIG

Distilled from docs.median.co (appending `.md` to page URLs returns markdown). All call signatures, parameters, and JSON shapes below are verbatim from the docs as of fetch date 2026-08-20. Doc quirks are flagged inline rather than "fixed" — where the docs contain a likely typo (e.g. `median.adjust.intialize`), the verbatim spelling is preserved.

JS Bridge is invoked via the global `median` object (or `import Median from "median-js-bridge"` NPM package where noted). Bridge calls should be made after the app/bridge is ready (e.g. after `deviceready`). *(Correction, added 2026-09-15: the `deviceready` mention was an authoring gloss, not docs content — it is a Cordova lifecycle event absent from Median's readiness model. The correct readiness patterns are `median_library_ready()` / `Median.onReady()`. The skills no longer carry this error; this note preserves the provenance record.)*

---

# SECTION 1 — NOTIFICATIONS (OneSignal-centric)

## Push Notifications Overview (topic)
- Purpose: Landing page for push plugins (OneSignal, Braze, Klaviyo, etc.) and push permission requirements.
- Key JS bridge calls referenced: `median.onesignal.login()` (associate users with external identifiers), `median.onesignal.info()` (retrieve OneSignal user and device data).
- Key terms (verbatim):
  - **APNs**: Apple's push service. Requires .p8 token key (recommended) or .p12 certificate, uploaded to OneSignal under Settings > Push & In-App > Apple iOS (APNs).
  - **FCM**: Google's push service. Requires Firebase project + Service Account JSON credentials uploaded to OneSignal under Settings > Push & In-App > Google Android (FCM).
  - **App ID / REST API Key**: App ID identifies app to SDK; REST API Key authorizes server-side delivery.
  - **Legacy Mode**: compatibility mode for apps built before Median adopted OneSignal SDK v5+; device-centric data model (see onesignal-legacy-mode page).
- Platform notes:
  - iOS: 'Push Notifications' capability required. App Studio builds with App Store Connect API workflow can register the Bundle ID + capabilities automatically. Manual signing/source builds: enable capability in Apple Developer account/Xcode.
  - Android: no additional steps beyond user consent.
- Gotchas: OneSignal is Median's default push provider (free tier, unlimited pushes). Push doesn't work on most simulators — test on physical devices. PWA push on iOS requires manual Add-to-Home-Screen and iOS 16.4+.
- Source: https://docs.median.co/docs/push-notifications-overview.md

## median.onesignal.* (plugin setup topic)
- Purpose: Connect OneSignal native iOS/Android SDKs to your app via the JS Bridge; SDK v5+ user-centric model by default.
- Setup: App Studio → Native Plugins > OneSignal → enter OneSignal App ID → save & rebuild. App initializes OneSignal on launch and prompts for push permission on first open (by default).
- Credentials table (verbatim):

  | Key | Where it's used |
  | --- | --- |
  | App ID | App Studio setup, REST API calls |
  | REST API Key | Server-side programmatic notifications |

- Platform notes (iOS source builds): Push Notifications + Background Modes (Remote notifications) capabilities required. App Groups (`group.YOUR_BUNDLE_IDENTIFIER.onesignal`) on BOTH main app target and OneSignalNotificationServiceExtension target required for badge counts, rich images, confirmed delivery.
- Gotchas:
  - Confirmed delivery analytics require an eligible (paid) OneSignal plan — not available on free plans even with App Groups configured.
  - REST API Key must stay server-side.
  - Apps built before OneSignal SDK v5 adoption: check Legacy Mode. Badge count management NOT available on legacy plugin (iOS v3 SDK / Android v4 SDK).
- Source: https://docs.median.co/docs/onesignal.md

## median.onesignal.register()
- Purpose: Trigger the native push permission prompt at a chosen moment (after disabling auto-register).
- Call signature: `median.onesignal.register();`
- Platform notes: Prompt-delaying applies to iOS and Android 13+ only; Android ≤12 grants push permission at install time (no prompt).
- Gotchas: Call only once per session; repeated calls after the user responded won't re-show the prompt. Disabling auto-register (App Studio → Native Plugins > OneSignal → disable "Auto-register for push notifications") still lets OneSignal initialize in background and generate a `oneSignalUserId` — for full GDPR hold use privacy consent instead.
- Minimal verified snippet:
  ```html
  <a onclick="median.onesignal.register()">Enable push notifications</a>
  ```
- Source: https://docs.median.co/docs/user-consent-management.md

## median.onesignal.userPrivacyConsent.grant() / .revoke()
- Purpose: GDPR-style consent gate — prevents OneSignal from initializing/transmitting ANY data until consent is granted.
- Call signature:
  ```javascript
  median.onesignal.userPrivacyConsent.grant();
  median.onesignal.userPrivacyConsent.revoke();
  ```
- Setup: App Studio → Native Plugins > OneSignal → enable "Require user privacy consent before transmitting data".
- Gotchas: Stricter than delayed registration — with consent required, OneSignal doesn't initialize at all (no `oneSignalUserId` generated until grant). Revoking consent stops data transmission but does NOT stop push delivery to an already-opted-in device — use Data Tags or `median.onesignal.logout()` for that.
- Source: https://docs.median.co/docs/user-consent-management.md

## Soft prompt pattern (topic)
- Purpose: In-app message shown BEFORE the native permission dialog to maximize opt-in.
- How: OneSignal In-App Message (HTML Composer in OneSignal dashboard or API). "Allow" button triggers `median.onesignal.register()`; "Maybe later" closes without native prompt.
- Platform notes: iOS shows the native "Allow notifications?" dialog only once — after "Don't Allow", the user must change it in Settings. Android 13+ recovery paths differ by OS/manufacturer.
- Source: https://docs.median.co/docs/user-consent-management.md

## median.onesignal.login(externalId)
- Purpose: Associate your own user identifier (email, account ID, any unique string) with the OneSignal user record (`oneSignalId`); enables cross-device targeting and reinstalls.
- Call signature: `median.onesignal.login("user@domain.com");`
- Parameters: single string argument — your external ID.
- External ID limits (verbatim): rejected values — `NA`, `NULL`, `null`, `none`, `not set`, `unknown`, `undefined`, `0`, `1`, `-1`, `NaN`, `00000000-0000-0000-0000-000000000000`, `-`, `ok`, `all`, `123ABC`, `UNQUALIFIED`, `INVALID_USER`. Max length 128 characters. Submitting a rejected value fails the assignment and leaves the User in its previous state.
- Gotchas: Call as soon as the user authenticates (login confirmation page or right after session check) — don't wait for a user action.
- Source: https://docs.median.co/docs/identifying-targeting-users.md

## median.onesignal.logout()
- Purpose: Disassociate the device from the OneSignal user record on app logout.
- Call signature: `median.onesignal.logout();`
- Gotchas: After logout, notifications sent to that `externalId` won't reach the device until `login()` again — prevents users on shared devices receiving each other's notifications.
- Source: https://docs.median.co/docs/identifying-targeting-users.md

## oneSignalInfo object + retrieval (median.onesignal.info / median.onesignal.onesignalInfo / median_onesignal_info)
- Purpose: Retrieve identifiers + subscription status needed for backend targeting.
- Data structure (SDK v5+, verbatim):

  | Field | Description |
  | --- | --- |
  | `oneSignalId` | OneSignal's internal user identifier |
  | `externalId` | Your identifier, assigned via `login()` |
  | `subscription.id` | Device-level push subscription identifier |
  | `subscription.token` | Push token for the device |
  | `subscription.optedIn` | `true` if the user has opted into push notifications |
  | `requiresUserPrivacyConsent` | `true` if consent hasn't been granted yet |

- Three retrieval methods (verbatim):
  1. **Automatic callback** — must be defined synchronously at page load (cannot be async/deferred):
     ```javascript
     function median_onesignal_info(data) {
         console.log(data.oneSignalId);
         console.log(data.subscription.id);
     }
     ```
  2. **Manual invocation**: `median.onesignal.info({ callback: "median_onesignal_info" });`
  3. **Promise-based**: `median.onesignal.onesignalInfo().then(...)` or `await median.onesignal.onesignalInfo()` — most flexible for SPAs.
- Verified login-page pattern:
  ```javascript
  async function loginUserAndPostOSId() {
    await median.onesignal.login("user@domain.com");
    const osInfo = await median.onesignal.info();
    $.ajax({ url: updateUserRecord, type: "POST", data: {
      yourAppUserId: userId,
      oneSignalUserId: osInfo.oneSignalId,     // user-specific, changes after login
      oneSignalExternalId: osInfo.externalId,  // same as userId
      oneSignalSubscriptionId: osInfo.subscription.id // device-specific, stable
    }, contentType: "application/json" });
  }
  ```
- When to use each identifier (verbatim):
  - `oneSignalId` — target a user across all devices, no own identifier system.
  - `externalId` — you've called `login()`; OneSignal manages user-device mapping. Right choice for most apps.
  - `subscription.id` — target one specific device; persists across logins, stable per device install.
- Demo: https://median.dev/onesignal/
- Source: https://docs.median.co/docs/identifying-targeting-users.md

## median.onesignal.tags.setTags / getTags / deleteTags
- Purpose: Manage OneSignal data tags (name-value string pairs for segmentation) from the web layer.
- Call signatures (verbatim):
  ```javascript
  median.onesignal.tags.setTags({ tags: { category: "sports", plan: "free", lastSeen: "2024-01-15" } })
    .then(function (tagResult) { console.log(tagResult); /* { success: true } */ });
  median.onesignal.tags.setTags({ tags: onesignalTags.tags, callback: tagSetCallbackFunction });

  median.onesignal.tags.getTags().then(function (tagResult) {
    console.log(tagResult); // { success: true, tags: { ... } }
  });
  median.onesignal.tags.getTags({ callback: tagGetCallbackFunction });

  median.onesignal.tags.deleteTags({ tags: ["category", "lastSeen"] }).then(...); // specific keys
  median.onesignal.tags.deleteTags().then(...);                                   // all tags
  ```
- Callback/promise shape: all return `{ success: true }`; `getTags` also includes a `tags` object. Both promise and callback patterns supported.
- Parameters: `setTags`/`deleteTags` take `{ tags: ... }` (object for set, array of key strings or omitted for delete); optional `callback` function.
- Alternative: Native UI — host a JSON file (see https://median.dev/onesignal/tags.json), set "Data Tags Native UI JSON URL" in App Studio, then `median.onesignal.showTagsUI();`
- Source: https://docs.median.co/docs/identifying-targeting-users.md

## Programmatic Notifications (OneSignal REST API — server-side)
- Purpose: Server-triggered transactional/behavioral push via OneSignal REST API (NOT a JS bridge API; sent from your backend).
- Endpoint: `POST https://api.onesignal.com/notifications` with header `Authorization: Key {REST_API_KEY}`.
- Targeting fields (verbatim):

  | Identifier | API field | Notes |
  | --- | --- | --- |
  | Your own user ID (email, account ID) | `include_aliases.external_id` | Recommended — maps directly to your user database |
  | OneSignal's internal user ID | `include_aliases.onesignal_id` | Use if you store `oneSignalId` in your backend |
  | A custom alias | `include_aliases.[alias_name]` | For advanced multi-ID targeting setups |

  Segments: `included_segments: ["Active Users"]`, `excluded_segments: ["Already Purchased"]`.
  Single device: `include_subscription_ids: ["device-subscription-id-here"]` (subscription.id; only changes on uninstall/reinstall).
- Verified full example:
  ```shell
  curl --location 'https://api.onesignal.com/notifications' \
  --header 'Authorization: Key {{YOUR...EY}}' \
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
- Key fields: `target_channel` always `"push"`; `"en"` required in headings/contents; `data.targetUrl` = deep-link URL opened inside app on tap; `include_aliases.external_id` accepts multiple users per request.
- Gotchas: REST API Key is a server-side secret — never in client JS or app code.
- Source: https://docs.median.co/docs/programmatic-notifications.md

## median.onesignal.badgeCount.set(n)
- Purpose: Set or clear the app icon badge count.
- Call signature:
  ```javascript
  median.onesignal.badgeCount.set(5);  // set
  median.onesignal.badgeCount.set(0);  // clear
  ```
- Platform notes: iOS natively supported on all devices; Android depends on OEM, not universally available. iOS auto-reset on notification open requires App Groups configured (both targets, same `group.YOUR_BUNDLE_IDENTIFIER.onesignal` identifier).
- Gotchas: NOT available on legacy OneSignal plugin (iOS v3 SDK / Android v4 SDK).
- Source: https://docs.median.co/docs/notification-customization.md

## Notification customization (icons & sounds — topic)
- Purpose: Configure notification icon, sound, badges.
- Icons: iOS uses app icon automatically (no per-notification customization). Android requires monochromatic icon w/ transparent background, uploaded in App Studio > Native Plugins > OneSignal. Icons are compiled into the native build — a non-conforming icon (e.g. full-color logo) renders as a solid white/colored square on some devices.
- Sounds: upload in App Studio under Native Plugins > OneSignal > Settings. Auto-named `custom_sound_1`, `custom_sound_2`, ... Median converts formats automatically.

  | Platform | Stored format | How to reference in API |
  | --- | --- | --- |
  | iOS | `.caf` | Include the extension: `custom_sound_1.caf` |
  | Android | `.mp3` | No extension needed: `custom_sound_1` |

- REST API sound fields (verbatim): `"ios_sound": "custom_sound_1.caf"`, `"android_channel_id": "{{YOUR_ANDROID_CHANNEL_ID}}"`. Android custom sounds are tied to notification channels — create the channel with the sound first, then use its ID.
- Source: https://docs.median.co/docs/notification-customization.md

## median_onesignal_push_opened(data) — notification tap handler
- Purpose: Global JS function the app calls automatically when a push notification is opened; receives the notification's Additional Data key-value pairs.
- Call signature: define on your page:
  ```javascript
  function median_onesignal_push_opened(data) {
      console.log(JSON.stringify(data));
  }
  // Expected output
  { 'airport': 'sfo', 'direction': 'outbound', 'dateRange': 'week' }
  ```
- Parameters: `data` = object of whatever custom key-value pairs were placed in the notification's Additional Data section (dashboard or REST API `data` field).
- Tap routing logic (verbatim decision flow): if payload contains `targetUrl` → navigate inside app; custom data present → call JS handler; neither → open app home screen.
- Gotchas:
  - `targetUrl` vs Launch URL: `targetUrl` (Additional Data) opens the URL INSIDE your app with full JS Bridge support — correct choice. Launch URL (composer field) opens in a popup browser window with NO JS Bridge access.
  - `targetUrl` is a RESERVED key — including it auto-triggers in-app navigation. For conditional navigation from your own JS, use a different key name (e.g. `openUrl`, `deepLink`) and navigate yourself inside `median_onesignal_push_opened`.
- Source: https://docs.median.co/docs/handling-notifications-taps.md

## median.onesignal.enableForegroundNotifications(boolean)
- Purpose: Control whether push notifications display while the app is open and in focus (suppressed by default).
- Call signature:
  ```javascript
  median.onesignal.enableForegroundNotifications(true);   // show when app open
  median.onesignal.enableForegroundNotifications(false);  // suppress (default)
  ```
- Gotchas: Callable from any page; takes effect immediately, active for the current session, no rebuild needed. A build-time default can be set in App Studio (Native Plugins > OneSignal); the runtime method overrides it.
- Source: https://docs.median.co/docs/foreground-notifications.md

---

# SECTION 2 — AUTH

## Authentication overview (topic)
- Purpose: Landing page for auth plugins (Social Login, Face ID/Touch ID/Android Biometric, Auth0, Passkeys, Clerk).
- Key bridge calls cited: `median.socialLogin.login()` (initiate social login), `median.auth.authenticate()` (prompt biometric authentication), `median.auth0.login()` (launch Auth0 Universal Login).
- Gotchas: IETF BCP (RFC 8252) requires user login in a browser session facilitated by a native app, NOT in an embedded webview — Google, Facebook, and Auth0 hosted login make this mandatory. Median's Social Login and Auth0 plugins use native SDKs and are the compliant paths.
- Source: https://docs.median.co/docs/authentication.md

## Face ID & Touch ID / Android Biometric (auth topic)
- Purpose: Biometric auth plugin overview; the page itself is a router — details live on `apple-face-id-touch-id` and `android-biometric-auth` pages (NOT fetched in this batch).
- Bridge call (from authentication overview): `median.auth.authenticate()`.
- Source: https://docs.median.co/docs/auth.md

## median.clerk.presentSignIn() / signOut() / getAuthStatus() / initialize()
- Purpose: Native Clerk sign-in/sign-up UI with JWT session token management surfaced to the web layer.
- Setup: Clerk publishable key (`pk_test_`/`pk_live_`) in App Studio Native Plugins (Advanced Mode).
- Call signatures + result objects (verbatim):
  ```javascript
  const result = await median.clerk.presentSignIn();
  // result object
  {
    state: "signedIn" | "signedOut",
    userId: "user_2abc...",           // present when signed in
    hasValidToken: true | false,
    token: "eyJ..."                   // present when hasValidToken is true
  }
  ```
  ```javascript
  median.clerk.presentSignIn({ callback: function(result) { ... } });  // callback form
  ```
  ```javascript
  const result = await median.clerk.signOut();
  {
    success: true | false,
    error: {                // present on failure only
      code: "NOT_INITIALIZED" | "SDK_ERROR",
      message: "..."
    }
  }
  ```
  ```javascript
  const status = await median.clerk.getAuthStatus();
  if (status.state === 'signedIn' && status.hasValidToken) {
    fetch('https://api.example.com/data', { headers: { Authorization: 'Bearer ' + status.token } });
  }
  ```
- Error codes (verbatim): `NOT_INITIALIZED` — Clerk SDK has not been initialized. `SDK_ERROR` — an unexpected error occurred within the Clerk SDK.
- Platform notes / CONTRADICTION FLAG: the "Auto-Initialization" callout says the Clerk SDK is automatically initialized at app startup from the app configuration, BUT the troubleshooting section states: "On iOS, the Clerk SDK is not initialized automatically at startup — you must call `median.clerk.initialize({ publishableKey: 'pk_...' })` from JavaScript before calling `presentSignIn`, `signOut`, or `getAuthStatus`. On Android, this call is optional because the SDK initializes from the server configuration." Treat iOS auto-init as unreliable; call `initialize()` on iOS and check it returns `success: true`.
- Gotchas: call `getAuthStatus()` immediately before each API request for a fresh token — never cache. A dismissed sign-in flow returns `state: "signedOut"` with no token. Demo: https://median.dev/clerk/
- Source: https://docs.median.co/docs/clerk.md

## median.auth0.login() / logout / status() / getCredentials() / renew()
- Purpose: Auth0 Universal Login via native Auth0 iOS/Android SDKs; optional refresh-token storage + biometric (Face ID/Touch ID/Android Biometric) re-auth.
- Plugin config (App Studio → Native Plugins → Advanced Mode, verbatim JSON):
  ```json
  {
    "domain": "dev-tax7avdd0xmeaavx.us.auth0.com",
    "clientId": "wTD****************KBwV",
    "scheme": "demo",
    "audience":"https://median-test.auth0.com/api/v2/"
  }
  // "scheme" is based on the URL scheme of your Android Callback URLs
  // "audience" is optional and based on your Auth0 tenant config
  ```
- Call signatures + response shapes (verbatim):
  ```javascript
  const credentials = await median.auth0.login({ scope: "email profile offline_access" });
  // credentials object
  {
    accessToken: STRING,
    idToken: STRING,
    scope: STRING,
    error: STRING, // if an error occurred
    refreshToken: STRING // with 'offline_access' scope configured
  }
  ```
  Logout (verbatim — note the docs show `median.auth0.logout.then(...)` with NO parentheses, likely a doc typo; treat logout as promise-returning):
  ```javascript
  median.auth0.logout.then(function(){
    error: STRING // if an error occurred
  })
  ```
  Biometrics + status:
  ```javascript
  median.auth0.login({ enableBiometrics: true, callback: median_auth0_post_login });

  const auth0Status = await median.auth0.status();
  // hasValidCredentials indicates whether credentials are stored,
  // and that the access token has not expired.
  {
    biometryAvailable: boolean,
    biometryType: 'touchId' | 'faceId' | 'none',
    hasValidCredentials: boolean
  }
  ```
  Saved credentials (auto-renews using stored refreshToken):
  ```javascript
  const credentials = await median.auth0.getCredentials();
  // { accessToken, idToken, scope, error?, refreshToken }
  ```
  Renew (verbatim — docs show `const credentials median.auth0.renew(...)` missing `=`; signature preserved):
  ```javascript
  const credentials = median.auth0.renew({
    refreshToken?: string // optional; defaults to saved token
  });
  ```
- Parameters: `login` accepts `scope` (optional string) and `enableBiometrics` (boolean) + `callback`; `renew` accepts optional `refreshToken`.
- Platform notes: biometrics cannot be tested on iOS/Android simulators. Requires a NEW Native Application in Auth0 for Universal Login. Callback/logout URLs of the form `demo://{domain}/android/{package}/callback`, `https://{domain}/ios/{bundle}/callback`, `{bundle}://{domain}/ios/{bundle}/callback`. Device Settings (Advanced Settings) must carry iOS bundle ID / Android package. Deep Linking + URL Scheme Protocol must be configured and match the Auth0 tenant.
- Gotchas: demo config is a sample — consult Auth0 experts for production. Demo: https://median.dev/auth0
- Source: https://docs.median.co/docs/auth0.md

## median.socialLogin.{facebook,google,apple}.login()
- Purpose: Trigger native Facebook Login / Google Sign-In / Sign In with Apple SDKs from the web layer.
- App Studio credentials (verbatim):

  | Provider | Parameters |
  | --- | --- |
  | Facebook | App ID, Display Name, Client Token |
  | Google | iOS Client ID, Android Client ID |
  | Apple | iOS Bundle ID |

- Call signatures (verbatim):
  ```javascript
  median.socialLogin.facebook.login({ 'callback' : <function>, 'scope' : '<text>', forceLimitedLogin: true | false, nonce: '<text>' });
  median.socialLogin.google.login({ 'callback' : <function> });
  median.socialLogin.apple.login({ 'callback' : <function>, 'scope' : '<text>' });
  ```
- Parameters (verbatim):
  - **callback** (required) — JS function invoked after login completes; receives token + user details.
  - **scope** (optional) — provider-specific scopes (Facebook/Google/Apple docs).
  - **forceLimitedLogin** (Facebook only) — true to force Limited Login even when ATT is enabled.
  - **nonce** (Facebook only) — string to verify authenticity in Limited Login mode.
- Response objects (verbatim):
  - Facebook success:
    ```json
    {
      "accessToken": "token string",
      "userId": "1234567890",
      "type": "facebook",
      "userDetails": {
        "userID": "<string>", "name": "<string>", "email": "<string>",
        "imageURL": "<string>", "friendIDs": "<string>", "birthday": "<string>",
        "ageRangeMin": "<number>", "ageRangeMax": "<number>",
        "hometown": "<string>", "location": "<string>", "gender": "<string>"
      },
      "authToken": "<limited-login-token>",
      "nonce": "<text>",
      "limitedLogin": true | false
    }
    ```
    Facebook error: `{ "error": "error description", "type": "facebook" }`
  - Google success: `{ "idToken": "token string", "type": "google" }`; error: `{ "error": "...", "type": "google" }`
  - Apple success: `{ "idToken": "token string", "code": "code string", "firstName": "first name", "lastName": "last name", "type": "apple" }`; error: `{ "error": "...", "type": "apple" }`
- Platform notes:
  - Facebook iOS: when ATT declined, "Limited Login" mode — no accessToken; you get `authToken` (a JWT, NOT usable for Graph API), a `nonce`, and `limitedLogin: true`. Validate per Facebook's Limited Login token docs.
  - Apple: firstName/lastName only returned on the FIRST authentication — persist them then; later logins give only JWT + user identifier (parse JWT for email, email_verified).
- Gotchas: detect Median vs browser via `navigator.userAgent.indexOf("median") >= 0` and swap browser-only/native-only buttons. Demo: https://median.dev/social-login/
- Source: https://docs.median.co/docs/social-login.md + https://docs.median.co/docs/social-login-javascript-callbacks.md

## Passkey Authentication / WebAuthn (topic)
- Purpose: Passwordless FIDO/WebAuthn login using native Face ID / Touch ID / Android biometrics.
- CRITICAL: "The WebAuthn / Passkey functionality operates entirely within your web implementation. It does not require the Median JavaScript Bridge." No `median.*` calls — your website's standard WebAuthn flow (challenge from server → device signs → verify server-side) is bridged to native hardware by the plugin.
- Setup:
  - Android: host `/.well-known/assetlinks.json` publicly with `delegate_permission/common.get_login_creds`; verify Android origin format `android:apk-key-hash:<BASE64URL SHA-256 fingerprint>`; include BOTH `requireResidentKey` (legacy, Android-required) and `residentKey` in `authenticatorSelection`; configure plugin in App Studio → Native Plugins > WebAuthn (Android) with `allowedUrls` (list of URL-pattern regexes allowed to create passkey requests).
  - iOS: native plugin NOT required; extend `/apple-app-site-association` with a `webcredentials` object:
    ```json
    { "webcredentials": { "apps": ["TEAMID.co.median.example"] } }
    ```
- Gotchas: AASA/assetlinks files must be publicly accessible (200, no redirects, `Content-Type: application/json`, https). Deep links/WebAuthn on iOS need a PATH in the URL (`http://example.com` fails, `http://example.com/PATH` works). Each subdomain needs its own entry + file. Server must implement `/passkey/register` and `/passkey/login` challenge endpoints. Demo: https://median.dev/passkey
- Source: https://docs.median.co/docs/passkey-authentication.md

---

# SECTION 3 — ANALYTICS / MONETIZATION

## Analytics overview (topic)
- Purpose: Landing page for analytics plugins (Firebase Analytics, AppsFlyer, Adjust, Meta App Events, Crashlytics, Branch, Intent Edge).
- Key bridge calls cited: `median.firebaseAnalytics.event.logEvent()`, `median.firebaseAnalytics.event.setUser()`, `median.facebook.events.send()` (Meta — separate page).
- Setup pattern: enable plugin in App Studio Native Plugins; upload config files (e.g. GoogleService-Info.plist) under Build & Deploy > Google Services.
- Source: https://docs.median.co/docs/analytics.md

## median.adjust.intialize / trackEvent / attributionInfo
- Purpose: Adjust SDK init, event tracking (incl. revenue), and attribution retrieval.
- NOTE ON SPELLING: the docs consistently spell the init function `intialize` (missing 'i') — verbatim, not a transcription error here. Use `median.adjust.intialize(...)`.
- Call signatures (verbatim):
  ```javascript
  // Manual init; optional boolean enables/disables SKAN (iOS only, ignored on Android)
  median.adjust.intialize(true | false);

  const adjustEvent = new AdjustEvent("abc123");   // event token from Adjust dashboard
  adjustEvent.setRevenue(0.01, "EUR");
  median.adjust.trackEvent(adjustEvent);

  median.adjust.attributionInfo();
  ```
- Parameters (verbatim):

  | Parameter | Type | Required | Description |
  | --- | --- | --- | --- |
  | SKAN enabled flag (intialize) | `boolean` | No | When Manual initialization is selected, enables or disables StoreKit Ad Network Attribution on iOS for that session. |
  | `adjustEvent` (trackEvent) | `AdjustEvent` | Yes | Instance created with your Adjust event token; optional revenue via `setRevenue(amount, currency)`. |

- Return: `attributionInfo()` returns attribution information stored by Adjust for the current user/install.
- App Studio settings: App Token (from Adjust dashboard AppView → app → App information → App details), Environment (**Production**/Development — set Production before publishing), Initialization (**Automatic**/Manual), StoreKit Ad Network Attribution (Enable/Disable, iOS).
- Gotchas: only call `intialize` when Initialization=Manual. Event tokens are created in the Adjust dashboard, not Median. `setRevenue(amount, currency)` needs a 3-letter currency code. Organic installs may return organic/limited attribution data. Test on real devices.
- Source: https://docs.median.co/docs/adjust.md

## median.appsflyer.setCustomerUserId / logEvent / getConversionData / getDeepLinkResult + callbacks
- Purpose: AppsFlyer user association, custom event logging, and conversion/deep-link data.
- Call signatures (verbatim, promise + NPM forms):
  ```javascript
  const { success } = await median.appsflyer.setCustomerUserId("user123");
  const { success } = await median.appsflyer.logEvent("purchase", { value: 99.99, currency: "USD" });

  const conversion = await median.appsflyer.getConversionData();
  const deepLink = await median.appsflyer.getDeepLinkResult();
  ```
  NPM package: `import Median from "median-js-bridge";` then `Median.appsflyer.setCustomerUserId(...)`, `Median.appsflyer.logEvent(...)`.
- Parameters (verbatim):

  | Parameter | Type | Required | Description |
  | --- | --- | --- | --- |
  | `userId` | `string` | Yes | Your unique customer or user |
  | `eventName` | `string` | Yes | Event name (e.g. `'purchase'`, `'signup'`) |
  | `eventValues` | `object` | No | Key-value pairs to attach to the event |

- Response shapes (verbatim):
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
- Global callback functions (verbatim):
  ```javascript
  function median_appsflyer_cd_success(conversionDataMap) { /* legacy install/conversion */ }
  function median_appsflyer_cd(data) { /* new conversion data */ }
  function median_appsflyer_deeplink_result(data) { /* UDL deep link; data.deeplinkValue */ }
  ```
  NPM listeners: `Median.appsflyer.conversionData.addListener(fn)`, `Median.appsflyer.deeplinkResult.addListener(fn)`, `Median.appsflyer.sdkStart.addListener(fn)` — each returns an ID for `removeListener(id)`.
- Platform notes: `af_status` is "Organic" or "Non-organic". iOS 14.5+ requires ATT (configurable ATT prompt delay in plugin). getConversionData/getDeepLinkResult are cache reads — "they never wait for the SDK, so on a cold launch they can resolve with `data: null`". Requires AppsFlyer plugin 1.4.0 (iOS) / 1.3.0 (Android); NOT yet in the NPM package.
- Gotchas: unset campaign fields come back as empty strings, not omitted; deep-link payload shape differs by platform — "Read `deeplinkValue` defensively rather than branching on the shape." Config: Android Dev Key, Apple Dev Key, Apple App ID in Native Plugins > AppsFlyer > Settings. Demo: https://median.dev/appsflyer/
- Source: https://docs.median.co/docs/appsflyer.md

## median.firebaseAnalytics.event.* (setConsent / collection / setUser / setUserProperty / defaultEventParameters / logEvent / logScreen / eCommerce)
- Purpose: Firebase Analytics runtime control — consent, collection toggle, user identity, events, screens, eCommerce.
- Setup: upload `google-services.json` (Android) + `GoogleService-Info.plist` (iOS) in App Studio Build & Deploy > Google Services; set default consent (Follow EU Consent Policy / Deny / Allow); enable plugin.
- Call signatures + parameters (verbatim):
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
- Parameter tables (verbatim):

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

  ProductItem optional fields (all optional; recommend at least `item_id` or `item_name`):
  ```javascript
  String item_id; String item_name; String item_category; String item_variant;
  String item_brand; String item_list_name; String item_list_id; double price;
  ```
- Gotchas: logging enabled by default; `collection({enabled:false})` disables ALL logging incl. automatic events and persists until re-enabled. setUser only after authentication; fire a follow-up event so identity/properties attach. Firebase auto-logs screen_view, session_start, first_open_time by default. Demo: https://median.dev/firebase-analytics/
- Source: https://docs.median.co/docs/firebase-analytics.md

## In-App Purchases (iap topic — router page)
- Purpose: Full StoreKit (Apple) and Google Play Billing support for consumables, non-consumables, subscriptions, one-time purchases.
- NOTE: the iap page is a router — implementation detail lives on `apple-iap` and `google-iap` pages (NOT fetched in this batch; fetch them for the actual bridge API).
- Gotchas: Apple mandates IAP for digital goods (consumables, non-consumables, non-renewing and auto-renewing subscriptions) inside iOS apps; follow both stores' payment guidelines (see faq-publishing#how-can-i-accept-payments-within-my-app).
- Source: https://docs.median.co/docs/iap.md

---

# SECTION 4 — APP CONFIG

## Native Plugins licensing (topic)
- Purpose: How plugins are licensed/trialed — the gotcha layer for every plugin above.
- Key facts (verbatim): "Native Plugins are available based on the tier of your license. Some plugins are only available with a Business or Enterprise plan." Essential and Plus plugins are trialable: Native Plugins section → click + next to the plugin → "Trial Active" label if unlicensed. Trial apps show a 'This app was developed using Median' popup until licensed. Some plugins require active third-party subscriptions. Median supports private/custom plugins (contact sales). Custom plugin development can integrate any third-party SDK into the App Studio build platform.
- Source: https://docs.median.co/docs/native-plugins-overview.md

## appConfig.json (topic)
- Purpose: The core key-value datastore for the app — branding assets, interface settings, native plugin configurations.
- Editing: App Studio UI or direct source edit. Import full config (incl. assets) from another app: App Studio > Build & Deploy > App Configuration > Import from existing app.
- Gotchas:
  - Invalid JSON in appConfig.json can crash the app AT LAUNCH — "Use a JSON validator or linter before saving changes."
  - To duplicate an app, use the "Clone" top-menu feature rather than config import (safer).
  - Plugin settings changed in appConfig.json/App Studio require a save + REBUILD of the app to take effect (per the OneSignal and Firebase pages).
- Source: https://docs.median.co/docs/app-configuration-appconfigjson.md

## Deep Linking (Universal Links / App Links / URL schemes — topic)
- Purpose: Let http(s) links open in the app instead of the browser; custom URL schemes for auth redirects.
- Android App Links: host `/.well-known/assetlinks.json` (proves domain control) BEFORE publishing; verify each hostname with Google's Statement List Generator. Example (verbatim):
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
- iOS Universal Links: host AASA at `/apple-app-site-association` or `/.well-known/apple-app-site-association`; explicit app ID + Associated Domains in provisioning profile; appID in file uses TEAMID. Example (verbatim):
  ```json
  {
    "applinks": {
      "apps": [],
      "details": [ { "appID": "TEAMID.co.median.example", "paths": ["*"] } ]
    }
  }
  ```
- iOS gotchas (verbatim): deep links require a PATH (`http://example.com` won't work, `http://example.com/PATH` will); every subdomain needs its own Associated Domains entry AND its own AASA file; serve with `Content-Type: application/json`, https + valid cert, HTTP 200, no redirects.
- Custom URL schemes (advanced): format `youruniquestring://`; app recognizes URLs like `youruniquestring.https://example.com/path`. Lowercase only — "Uppercase letters and numbers may lead to inconsistent behavior across platforms." Not supported in desktop browsers. Commonly used for mobile auth redirects (e.g. Auth0 callback scheme).
- Best practice: add both root domain and `www` host. Validator: deep-linking-validator page. Demo: https://median.dev/deep-linking/
- Source: https://docs.median.co/docs/deep-linking.md

## median.downloads.init / downloadFile / showUI (Offline Download Manager)
- Purpose: Download documents/media for offline access; built-in file manager UI + programmatic control.
- Call signatures (verbatim):
  ```javascript
  median.downloads.init({ callback: downloadCallback }); // returns promise — register before any downloads
  median.downloads.downloadFile({ url: "URL", title: "Title" });
  median.downloads.showUI();
  ```
- Parameters (verbatim): `url` (required — must start with http/https), `title` (required — name shown in UI), `identifier` (optional — string to differentiate simultaneous downloads in the callback), `details` (optional — description shown below title), `date` (optional — yyyy-mm-dd shown below details, e.g. podcast publish date).
- Callback shape (verbatim): callback receives `{ identifier, event }` where event is `"progress"`, `"done"`, or `"error"`.
  - progress: also `bytesWritten` (bytes downloaded), `expectedBytes` (server-indicated size)
  - error: also `errorMessage` (string reason)
- Gotchas: configure an offline.html page in the app so the download manager UI can open when the app is offline for extended periods (see Offline Page docs + Codepen PoJbEEN sample). Demo: https://median.dev/offline-download-manager/
- Source: https://docs.median.co/docs/offline-download-manager.md

---

## Coverage

Fetched OK (24/24):
1. https://docs.median.co/docs/push-notifications-overview.md
2. https://docs.median.co/docs/onesignal.md
3. https://docs.median.co/docs/user-consent-management.md
4. https://docs.median.co/docs/identifying-targeting-users.md
5. https://docs.median.co/docs/programmatic-notifications.md
6. https://docs.median.co/docs/notification-customization.md
7. https://docs.median.co/docs/handling-notifications-taps.md
8. https://docs.median.co/docs/foreground-notifications.md
9. https://docs.median.co/docs/authentication.md
10. https://docs.median.co/docs/auth.md
11. https://docs.median.co/docs/clerk.md (succeeded on retry — rate limit on first attempt)
12. https://docs.median.co/docs/auth0.md
13. https://docs.median.co/docs/social-login.md
14. https://docs.median.co/docs/social-login-javascript-callbacks.md
15. https://docs.median.co/docs/passkey-authentication.md
16. https://docs.median.co/docs/analytics.md
17. https://docs.median.co/docs/adjust.md
18. https://docs.median.co/docs/appsflyer.md
19. https://docs.median.co/docs/firebase-analytics.md
20. https://docs.median.co/docs/native-plugins-overview.md
21. https://docs.median.co/docs/app-configuration-appconfigjson.md
22. https://docs.median.co/docs/deep-linking.md
23. https://docs.median.co/docs/offline-download-manager.md
24. https://docs.median.co/docs/iap.md

FETCH FAILED: none.

Notes on depth: `auth.md` (biometrics) and `iap.md` are router pages — actual bridge APIs live on `apple-face-id-touch-id`, `android-biometric-auth`, `apple-iap`, and `google-iap` pages (outside the assigned page list). OneSignal Legacy Mode (`onesignal-legacy-mode`) and Data Tags concept page (`data-tags`) are referenced but were not in scope.
