# OneSignal Push — Full API Reference (Median JS Bridge)

All call signatures, parameters, and JSON shapes verbatim from docs.median.co as of 2026-08-20. OneSignal is Median's default push provider (free tier, unlimited pushes). SDK v5+ user-centric model by default.

## Plugin setup & credentials

App Studio → Native Plugins > OneSignal → enter OneSignal App ID → save & rebuild. The app initializes OneSignal on launch and prompts for push permission on first open (by default).

| Key | Where it's used |
| --- | --- |
| App ID | App Studio setup, REST API calls |
| REST API Key | Server-side programmatic notifications (keep server-side) |

Push provider credentials (uploaded to OneSignal, not Median):
- **APNs** (iOS): .p8 token key (recommended) or .p12 certificate, under Settings > Push & In-App > Apple iOS (APNs).
- **FCM** (Android): Firebase project + Service Account JSON credentials, under Settings > Push & In-In-App > Google Android (FCM).

Platform notes:
- iOS: 'Push Notifications' capability required. App Studio builds with App Store Connect API workflow register the Bundle ID + capabilities automatically. Manual signing/source builds: enable the capability in Apple Developer account/Xcode.
- iOS source builds: Push Notifications + Background Modes (Remote notifications); App Groups (`group.YOUR_BUNDLE_IDENTIFIER.onesignal`) on BOTH the main app target and the OneSignalNotificationServiceExtension target for badge counts, rich images, confirmed delivery.
- Android: no additional steps beyond user consent.
- Legacy Mode: compatibility mode for apps built before Median adopted OneSignal SDK v5+; device-centric data model. Badge count management NOT available on the legacy plugin (iOS v3 SDK / Android v4 SDK).
- Confirmed delivery analytics require an eligible (paid) OneSignal plan — not available on free plans even with App Groups configured.
- Push doesn't work on most simulators — test on physical devices. PWA push on iOS requires manual Add-to-Home-Screen and iOS 16.4+.

## Permission & consent flow

### median.onesignal.register()
Trigger the native push permission prompt at a chosen moment (after disabling auto-register in App Studio → Native Plugins > OneSignal → disable "Auto-register for push notifications").

```html
<a onclick="median.onesignal.register()">Enable push notifications</a>
```

- Prompt-delaying applies to iOS and Android 13+ only; Android ≤12 grants push permission at install time (no prompt).
- Call only once per session; repeated calls after the user responded won't re-show the prompt.
- Disabling auto-register still lets OneSignal initialize in background and generate a `oneSignalUserId` — for full GDPR hold use privacy consent instead.

### median.onesignal.userPrivacyConsent.grant() / .revoke()
GDPR-style consent gate. Enable in App Studio → Native Plugins > OneSignal → enable "Require user privacy consent before transmitting data". Prevents OneSignal from initializing/transmitting ANY data until consent is granted.

```javascript
median.onesignal.userPrivacyConsent.grant();
median.onesignal.userPrivacyConsent.revoke();
```

- Stricter than delayed registration — with consent required, OneSignal doesn't initialize at all (no `oneSignalUserId` generated until grant).
- Revoking consent stops data transmission but does NOT stop push delivery to an already-opted-in device — use Data Tags or `median.onesignal.logout()` for that.

### Soft prompt pattern
In-app message shown BEFORE the native permission dialog to maximize opt-in. Built as a OneSignal In-App Message (HTML Composer in the OneSignal dashboard or API). "Allow" button triggers `median.onesignal.register()`; "Maybe later" closes without the native prompt.
- iOS shows the native "Allow notifications?" dialog only once — after "Don't Allow", the user must change it in Settings. Android 13+ recovery paths differ by OS/manufacturer.

## Identifying & targeting users

### median.onesignal.login(externalId) / logout()
```javascript
median.onesignal.login("user@domain.com");   // associate your user ID (email, account ID, any unique string)
median.onesignal.logout();                    // disassociate on app logout
```

- Call `login` as soon as the user authenticates (login confirmation page or right after session check) — don't wait for a user action.
- After `logout`, notifications sent to that `externalId` won't reach the device until `login()` again — prevents users on shared devices receiving each other's notifications.

External ID limits (verbatim):
- Rejected values: `NA`, `NULL`, `null`, `none`, `not set`, `unknown`, `undefined`, `0`, `1`, `-1`, `NaN`, `00000000-0000-0000-0000-000000000000`, `-`, `ok`, `all`, `123ABC`, `UNQUALIFIED`, `INVALID_USER`.
- Max length 128 characters.
- Submitting a rejected value fails the assignment and leaves the User in its previous state.

### oneSignalInfo object (SDK v5+)

| Field | Description |
| --- | --- |
| `oneSignalId` | OneSignal's internal user identifier |
| `externalId` | Your identifier, assigned via `login()` |
| `subscription.id` | Device-level push subscription identifier |
| `subscription.token` | Push token for the device |
| `subscription.optedIn` | `true` if the user has opted into push notifications |
| `requiresUserPrivacyConsent` | `true` if consent hasn't been granted yet |

Three retrieval methods:
1. **Automatic callback** — must be defined synchronously at page load (cannot be async/deferred):
   ```javascript
   function median_onesignal_info(data) {
       console.log(data.oneSignalId);
       console.log(data.subscription.id);
   }
   ```
2. **Manual invocation**: `median.onesignal.info({ callback: "median_onesignal_info" });`
3. **Promise-based**: `median.onesignal.onesignalInfo().then(...)` or `await median.onesignal.info()` — most flexible for SPAs.

Verified login-page pattern (post identifiers to your backend):
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

When to use each identifier:
- `oneSignalId` — target a user across all devices, no own identifier system.
- `externalId` — you've called `login()`; OneSignal manages user-device mapping. Right choice for most apps.
- `subscription.id` — target one specific device; persists across logins, stable per device install.

Demo: https://median.dev/onesignal/

### Data tags
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

- All return `{ success: true }`; `getTags` also includes a `tags` object. Both promise and callback patterns supported.
- `setTags`/`deleteTags` take `{ tags: ... }` (object for set, array of key strings or omitted for delete); optional `callback` function.
- Native UI alternative: host a JSON file (see https://median.dev/onesignal/tags.json), set "Data Tags Native UI JSON URL" in App Studio, then `median.onesignal.showTagsUI();`.

## Notification taps

### median_onesignal_push_opened(data)
Global JS function the app calls automatically when a push notification is opened; receives the notification's Additional Data key-value pairs.

```javascript
function median_onesignal_push_opened(data) {
    console.log(JSON.stringify(data));
}
// Expected output
{ 'airport': 'sfo', 'direction': 'outbound', 'dateRange': 'week' }
```

Tap routing logic (verbatim decision flow): if payload contains `targetUrl` → navigate inside app; custom data present → call JS handler; neither → open app home screen.

- `targetUrl` vs Launch URL: `targetUrl` (Additional Data) opens the URL INSIDE your app with full JS Bridge support — correct choice. Launch URL (composer field) opens in a popup browser window with NO JS Bridge access.
- `targetUrl` is a RESERVED key — including it auto-triggers in-app navigation. For conditional navigation from your own JS, use a different key name (e.g. `openUrl`, `deepLink`) and navigate yourself inside `median_onesignal_push_opened`.

## Badge count

```javascript
median.onesignal.badgeCount.set(5);  // set
median.onesignal.badgeCount.set(0);  // clear
```

- iOS natively supported on all devices; Android depends on OEM, not universally available.
- iOS auto-reset on notification open requires App Groups configured (both targets, same `group.YOUR_BUNDLE_IDENTIFIER.onesignal` identifier).
- NOT available on legacy OneSignal plugin (iOS v3 SDK / Android v4 SDK).

## Foreground notifications

```javascript
median.onesignal.enableForegroundNotifications(true);   // show when app open
median.onesignal.enableForegroundNotifications(false);  // suppress (default)
```

- Callable from any page; takes effect immediately, active for the current session, no rebuild needed. A build-time default can be set in App Studio (Native Plugins > OneSignal); the runtime method overrides it.

## In-app messages (median.onesignal.iam.*)

OneSignal in-app messages (IAM): rich UI panels shown inside the app without requiring the push permission prompt. Designed entirely in OneSignal's dashboard (HTML Composer); the bridge drives triggers, pause/resume, and click handling. Source: https://docs.median.co/docs/in-app-messages.md

```javascript
// Trigger management
median.onesignal.iam.addTrigger({ key: "value" });
median.onesignal.iam.addTriggers({ key1: "value1", key2: "value2" });
median.onesignal.iam.removeTriggerForKey("key");
median.onesignal.iam.getTriggerValueForKey("key");

// Pause and resume
median.onesignal.iam.pauseInAppMessages();
median.onesignal.iam.resumeInAppMessages();

// Click handler
median.onesignal.iam.setInAppMessageClickHandler("yourHandlerFunctionName");
```

| Call | Argument | Description |
| --- | --- | --- |
| `addTrigger` | `{ key: "value" }` | Add a single trigger; the dashboard-configured message displays when its trigger conditions match |
| `addTriggers` | `{ key1: "value1", key2: "value2" }` | Add multiple triggers at once |
| `removeTriggerForKey` | `"key"` | Remove a trigger by key |
| `getTriggerValueForKey` | `"key"` | Retrieve the current value of a trigger |
| `pauseInAppMessages` | — | Stop showing in-app messages (e.g. during checkout, video playback) |
| `resumeInAppMessages` | — | Resume showing in-app messages |
| `setInAppMessageClickHandler` | `"yourHandlerFunctionName"` (global fn name) | Runs when the user clicks an action button inside an IAM; called with the click-event data |

- IAMs display only while the user is actively using the app (unlike push).
- The OneSignal SDK delivers IAMs only after the user granted push notification permission — despite IAMs not needing the prompt themselves.
- Android virtual simulators cannot display IAMs (physical device required); iOS works on physical devices and the iOS simulator.
- The docs describe pause as "temporarily suppress" but do not document whether the paused state survives an app restart — verify on-device.
- Soft-prompt pattern: an IAM with "Allow" → `median.onesignal.register()` / "Maybe later" → dismiss (see Permission & consent flow above).

## Programmatic notifications (OneSignal REST API — server-side)

NOT a JS bridge API — sent from your backend. Endpoint: `POST https://api.onesignal.com/notifications` with header `Authorization: Key ***`.

Targeting fields:

| Identifier | API field | Notes |
| --- | --- | --- |
| Your own user ID (email, account ID) | `include_aliases.external_id` | Recommended — maps directly to your user database |
| OneSignal's internal user ID | `include_aliases.onesignal_id` | Use if you store `oneSignalId` in your backend |
| A custom alias | `include_aliases.[alias_name]` | For advanced multi-ID targeting setups |

- Segments: `included_segments: ["Active Users"]`, `excluded_segments: ["Already Purchased"]`.
- Single device: `include_subscription_ids: ["device-subscription-id-here"]` (subscription.id; only changes on uninstall/reinstall).

Verified full example:
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

Key fields: `target_channel` always `"push"`; `"en"` required in headings/contents; `data.targetUrl` = deep-link URL opened inside app on tap; `include_aliases.external_id` accepts multiple users per request. REST API Key is a server-side secret — never in client JS or app code.

## Icons & sounds

- **Icons**: iOS uses the app icon automatically (no per-notification customization). Android requires a monochromatic icon w/ transparent background, uploaded in App Studio > Native Plugins > OneSignal. Icons are compiled into the native build — a non-conforming icon (e.g. full-color logo) renders as a solid white/colored square on some devices.
- **Sounds**: upload in App Studio under Native Plugins > OneSignal > Settings. Auto-named `custom_sound_1`, `custom_sound_2`, ... Median converts formats automatically.

| Platform | Stored format | How to reference in API |
| --- | --- | --- |
| iOS | `.caf` | Include the extension: `custom_sound_1.caf` |
| Android | `.mp3` | No extension needed: `custom_sound_1` |

REST API sound fields: `"ios_sound": "custom_sound_1.caf"`, `"android_channel_id": "{{YOUR_ANDROID_CHANNEL_ID}}"`. Android custom sounds are tied to notification channels — create the channel with the sound first, then use its ID.

## Sources

- https://docs.median.co/docs/push-notifications-overview.md
- https://docs.median.co/docs/onesignal.md
- https://docs.median.co/docs/user-consent-management.md
- https://docs.median.co/docs/identifying-targeting-users.md
- https://docs.median.co/docs/programmatic-notifications.md
- https://docs.median.co/docs/notification-customization.md
- https://docs.median.co/docs/handling-notifications-taps.md
- https://docs.median.co/docs/foreground-notifications.md
- https://docs.median.co/docs/in-app-messages.md
