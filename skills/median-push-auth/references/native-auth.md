# Native Auth — Full API Reference (Median JS Bridge)

All call signatures, parameters, and response shapes verbatim from docs.median.co as of 2026-08-20. Doc quirks (typos, contradictions) are flagged inline and preserved verbatim rather than "fixed".

## Compliance context

IETF BCP (RFC 8252) requires user login in a browser session facilitated by a native app, NOT in an embedded webview — Google, Facebook, and Auth0 hosted login make this mandatory. Median's Social Login and Auth0 plugins use native SDKs and are the compliant paths.

Detect Median vs browser: `navigator.userAgent.indexOf("median") >= 0` — swap browser-only/native-only buttons accordingly.

## Clerk

Setup: Clerk publishable key (`pk_test_`/`pk_live_`) in App Studio Native Plugins (Advanced Mode).

### median.clerk.presentSignIn()
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
Callback form: `median.clerk.presentSignIn({ callback: function(result) { ... } });`
A dismissed sign-in flow returns `state: "signedOut"` with no token.

### median.clerk.signOut()
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
Error codes: `NOT_INITIALIZED` — Clerk SDK has not been initialized. `SDK_ERROR` — an unexpected error occurred within the Clerk SDK.

### median.clerk.getAuthStatus()
```javascript
const status = await median.clerk.getAuthStatus();
if (status.state === 'signedIn' && status.hasValidToken) {
  fetch('https://api.example.com/data', { headers: { Authorization: 'Bearer ' + status.token } });
}
```
Call immediately before each API request for a fresh token — never cache.

### median.clerk.initialize() — iOS initialization contradiction (flagged)
The docs contradict themselves:
- The "Auto-Initialization" callout says the Clerk SDK is automatically initialized at app startup from the app configuration.
- The troubleshooting section states: "On iOS, the Clerk SDK is not initialized automatically at startup — you must call `median.clerk.initialize({ publishableKey: 'pk_...' })` from JavaScript before calling `presentSignIn`, `signOut`, or `getAuthStatus`. On Android, this call is optional because the SDK initializes from the server configuration."

Treat iOS auto-init as unreliable; call `initialize()` on iOS and check it returns `success: true`.

Demo: https://median.dev/clerk/

## Auth0

Setup: App Studio → Native Plugins → Advanced Mode (verbatim JSON):
```json
{
  "domain": "dev-tax7avdd0xmeaavx.us.auth0.com",
  "clientId": "wTD****************KBwV",
  "scheme": "demo",
  "audience": "https://median-test.auth0.com/api/v2/"
}
// "scheme" is based on the URL scheme of your Android Callback URLs
// "audience" is optional and based on your Auth0 tenant config
```

Platform requirements:
- Requires a NEW Native Application in Auth0 for Universal Login.
- Callback/logout URLs of the form `demo://{domain}/android/{package}/callback`, `https://{domain}/ios/{bundle}/callback`, `{bundle}://{domain}/ios/{bundle}/callback`.
- Device Settings (Advanced Settings) must carry iOS bundle ID / Android package.
- Deep Linking + URL Scheme Protocol must be configured and match the Auth0 tenant.
- Biometrics cannot be tested on iOS/Android simulators.
- Demo config is a sample — consult Auth0 experts for production.

### median.auth0.login()
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
Parameters: `scope` (optional string), `enableBiometrics` (boolean), `callback`. Biometric (Face ID/Touch ID/Android Biometric) re-auth:
```javascript
median.auth0.login({ enableBiometrics: true, callback: median_auth0_post_login });
```

### median.auth0.logout() — doc typo flagged
Verbatim from the docs (note `median.auth0.logout.then(...)` with NO parentheses — likely a doc typo; treat logout as promise-returning):
```javascript
median.auth0.logout.then(function(){
  error: STRING // if an error occurred
})
```

### median.auth0.status()
```javascript
const auth0Status = await median.auth0.status();
// hasValidCredentials indicates whether credentials are stored,
// and that the access token has not expired.
{
  biometryAvailable: boolean,
  biometryType: 'touchId' | 'faceId' | 'none',
  hasValidCredentials: boolean
}
```

### median.auth0.getCredentials()
Saved credentials (auto-renews using stored refreshToken):
```javascript
const credentials = await median.auth0.getCredentials();
// { accessToken, idToken, scope, error?, refreshToken }
```

### median.auth0.renew() — doc typo flagged
Verbatim from the docs (`const credentials median.auth0.renew(...)` is shown missing `=`; signature preserved):
```javascript
const credentials = median.auth0.renew({
  refreshToken?: string // optional; defaults to saved token
});
```

Demo: https://median.dev/auth0

## Social Login (Facebook / Google / Apple)

App Studio credentials:

| Provider | Parameters |
| --- | --- |
| Facebook | App ID, Display Name, Client Token |
| Google | iOS Client ID, Android Client ID |
| Apple | iOS Bundle ID |

### Call signatures (verbatim)
```javascript
median.socialLogin.facebook.login({ 'callback' : <function>, 'scope' : '<text>', forceLimitedLogin: true | false, nonce: '<text>' });
median.socialLogin.google.login({ 'callback' : <function> });
median.socialLogin.apple.login({ 'callback' : <function>, 'scope' : '<text>' });
```

Parameters:
- **callback** (required) — JS function invoked after login completes; receives token + user details.
- **scope** (optional) — provider-specific scopes (Facebook/Google/Apple docs).
- **forceLimitedLogin** (Facebook only) — true to force Limited Login even when ATT is enabled.
- **nonce** (Facebook only) — string to verify authenticity in Limited Login mode.

### Response objects (verbatim)

Facebook success:
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

Google success: `{ "idToken": "token string", "type": "google" }`
Google error: `{ "error": "...", "type": "google" }`

Apple success: `{ "idToken": "token string", "code": "code string", "firstName": "first name", "lastName": "last name", "type": "apple" }`
Apple error: `{ "error": "...", "type": "apple" }`

Platform notes:
- Facebook iOS: when ATT declined, "Limited Login" mode — no accessToken; you get `authToken` (a JWT, NOT usable for Graph API), a `nonce`, and `limitedLogin: true`. Validate per Facebook's Limited Login token docs.
- Apple: firstName/lastName only returned on the FIRST authentication — persist them then; later logins give only JWT + user identifier (parse JWT for email, email_verified).

Demo: https://median.dev/social-login/

## Face ID / Touch ID / Android Biometric (secret storage)

One plugin ("Face ID / TouchID Android Biometrics") covers both platforms. It stores a secret — a single string defined by the website developer (credentials JSON, authentication token, or any other login data) — in hardware-backed storage (iOS Keychain; Android cryptographic hardware) so it cannot be retrieved without biometric authentication. Saving the secret does not require user interaction.

Flow (verbatim from both subpages): check biometric availability → after login, save a secret → on future visits, if a saved secret exists, prompt biometric authentication → use the recovered secret to log the user in or populate the login form.

The overview page (https://docs.median.co/docs/auth.md) is a router; the full API lives on the subpages (both fetched and verified):
- https://docs.median.co/docs/apple-face-id-touch-id.md
- https://docs.median.co/docs/android-biometric-auth.md

*(Note: an earlier revision of this skill cited `median.auth.authenticate()` from the docs overview — that call does not appear anywhere in the current docs; the API surface is `status`/`save`/`get`/`delete`.)*

Demo: https://median.dev/auth/

### median.auth.status()
```javascript
median.auth.status({
  'minimumAndroidBiometric': 'strong' | 'weak',  // optional, default 'strong' (Android)
  'callbackFunction': CALLBACK
});
```
The app returns a promise, or executes CALLBACK, with:
- `hasTouchId`: `true` | `false` — iOS: "Indicates if the device is running iOS 9+ and there are fingerprints enrolled, or FaceID is enabled" (covers Face ID too); Android: `true` on devices with fingerprints enrolled
- `biometryType`: `'touchId'`, `'faceId'`, or `'none'` — "populated on iOS only to differentiate between TouchID and FaceID"
- `hasSecret`: `true` | `false`

### median.auth.save()
```javascript
median.auth.save({
  'secret': secret,                              // single string: JSON credentials, auth token, etc.
  'minimumAndroidBiometric': 'strong' | 'weak',  // optional, default 'strong' (Android)
  'callbackFunction': CALLBACK                   // iOS docs: optional, "called after the save operation with a success parameter"
});
```
Android returns a promise, or executes CALLBACK, with an object with a `success` field. Typically called after checking `status()` first.

### median.auth.get()
```javascript
median.auth.get({
  'callbackFunction': CALLBACK,                  // iOS docs: required
  'minimumAndroidBiometric': 'strong' | 'weak',  // optional, default 'strong' (Android)
  'prompt': 'PROMPT',                            // optional; "Prompt is only available for iOS and it has been included above to ensure cross-platform compatibility"
  'callbackOnCancel': INTEGER                    // optional
});
```
The app returns a promise, or executes CALLBACK, with:
- `success`: `true` | `false`
- `error`: provided success is `false` (see error codes below)
- `secret`: the previously stored secret

`callbackOnCancel`: "If set to 1 and the user cancels the authentication, the callback will be run with `success=false`, `error=userCanceled`. If `callbackOnCancel` is not set (or set to 0), the callback will not be run."

### median.auth.delete()
```javascript
median.auth.delete({'callbackFunction': CALLBACK});
```
The app returns a promise, or executes CALLBACK, with `success` (`true` | `false`) and `error` (provided success is false). Docs pattern: "delete secret if credentials are incorrect" — if the retrieved secret fails to log the user in, delete it and allow manual login.

### Error codes (verbatim)
"In general, you will only need to handle authenticationFailed in the 'get secret' request."
- `duplicateItem`: Secret already exists
- `itemNotFound`: No secret saved
- `authenticationFailed`: Biometric check failed
- `genericError`: Unexpected failure
- `userCanceled`: User canceled the auth prompt
- `unimplemented`: Plugin not supported

### Docs quirks flagged
- The iOS page marks `callbackFunction` "required" for `status` and `get` while the Android page marks it "optional if callback is used rather than promise" — Android documents promise returns for every call; when targeting iOS, use the callback form for `status`/`get` per the iOS page.
- The iOS page's "Promise method" examples are syntactically malformed — they end `median.auth.status().then(function (result) { ... }` missing the closing `);` (preserved as a docs artifact; write your own promise calls with correct syntax).

### Whitelist domains
"By default, any page loaded in your app will be able to use Median JavaScript Bridge to retrieve secrets." If domains you do not control can load in your app, whitelist yours: App Studio → Native Plugins → Settings for the Face ID/TouchID Android Biometrics plugin. "Regular expressions are supported to allow wildcards in the URL."

### Testing
Physical devices recommended ("as well as older models or more restricted device configurations" — MDM may disable biometrics). Appetize simulators also work: open the demo page in the simulator, click the Face ID button (iOS) / Fingerprint ID button (Android) to enable biometrics, log in with credentials, then use the biometric login button without re-entering them.

## Passkeys / WebAuthn

CRITICAL: "The WebAuthn / Passkey functionality operates entirely within your web implementation. It does not require the Median JavaScript Bridge." No `median.*` calls — your website's standard WebAuthn flow (challenge from server → device signs → verify server-side) is bridged to native hardware by the plugin.

### Android setup
- Host `/.well-known/assetlinks.json` publicly with `delegate_permission/common.get_login_creds`.
- Verify Android origin format `android:apk-key-hash:<BASE64URL SHA-256 fingerprint>`.
- Include BOTH `requireResidentKey` (legacy, Android-required) and `residentKey` in `authenticatorSelection`.
- Configure the plugin in App Studio → Native Plugins > WebAuthn (Android) with `allowedUrls` (list of URL-pattern regexes allowed to create passkey requests).

### iOS setup
- Native plugin NOT required; extend `/apple-app-site-association` with a `webcredentials` object:
```json
{ "webcredentials": { "apps": ["TEAMID.co.median.example"] } }
```

### Gotchas
- AASA/assetlinks files must be publicly accessible (200, no redirects, `Content-Type: application/json`, https).
- Deep links/WebAuthn on iOS need a PATH in the URL (`http://example.com` fails, `http://example.com/PATH` works).
- Each subdomain needs its own entry + file.
- Server must implement `/passkey/register` and `/passkey/login` challenge endpoints.

Demo: https://median.dev/passkey

## Sources

- https://docs.median.co/docs/authentication.md
- https://docs.median.co/docs/auth.md
- https://docs.median.co/docs/apple-face-id-touch-id.md
- https://docs.median.co/docs/android-biometric-auth.md
- https://docs.median.co/docs/clerk.md
- https://docs.median.co/docs/auth0.md
- https://docs.median.co/docs/social-login.md
- https://docs.median.co/docs/social-login-javascript-callbacks.md
- https://docs.median.co/docs/passkey-authentication.md
