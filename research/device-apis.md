# Median.co JavaScript Bridge — Device APIs + Native Functionality (Domain Research)

Distilled from docs.median.co (fetched 2026-08-20, `.md` variants). Verbatim parameter names, defaults, and response shapes; no invented fields. Common prerequisite for all bridge calls: JavaScript Bridge enabled in App Studio, and the relevant **Native Plugin** toggled on where noted.

---

## median.deviceInfo (Device Info)
- **Purpose**: Returns real-time device, app, and advertising-identifier data for analytics, compatibility checks, support, and attribution.
- **Call signatures (all three work)**:
  - `median.run.deviceInfo()` — triggers manual call to your `median_device_info()` function
  - `median.deviceInfo().then(...)` / `await median.deviceInfo()` — promise
  - `function median_device_info(deviceInfo) {...}` — auto-invoked by the app **after every page load** (must be defined at page load time; cannot be added asynchronously/deferred)
- **Parameters**: none (input side); response payload fields below.
- **Response shape / example** (verbatim from docs):
  ```javascript
  {
    platform: 'ios',                                   // 'ios' | 'android'
    "SHA-1": 'XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX',    // only on Android (signing cert fingerprint)
    appId: 'io.median.example',
    appVersion: '1.0.0',
    appBuild: '1.0.0',                                 // appVersionCode (number) on Android
    carrierNames: ['AT&T'],                            // Android requires READ_PHONE_STATE
    distribution: 'release',
    hardware: 'armv8',
    installationId: 'xxxx-xxxx-xxxx-xxxx',
    apnsToken: '',                                     // only on iOS
    language: 'en',
    model: 'iPhone',
    os: 'iOS',
    osVersion: '10.3',
    timeZone: 'America/New_York',
    isFirstLaunch: false,                              // true on first launch of the app
    idfa: '00000000-0000-0000-0000-000000000000',      // iOS only; only if ad/attribution SDK configured
    gaid: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'      // Android only; only if ad/attribution SDK configured
  }
  ```
- **Field reference** (Key | Type | Platform | Description — verbatim): `platform` string Both (`ios`/`android`); `os` string Both; `osVersion` string Both; `appId` string Both; `appVersion` string Both; `appBuild` string/number Both (numeric `appVersionCode` on Android); `installationId` string Both; `distribution` string Both (e.g. `release`); `hardware` string Both (e.g. `armv8`); `model` string Both; `carrierNames` array Both (Android requires `READ_PHONE_STATE`); `timeZone` string Both; `language` string Both; `isFirstLaunch` boolean Both; `SHA-1` string Android-only; `apnsToken` string iOS-only; `idfa` string iOS-only (requires ad SDK); `gaid` string Android-only (requires ad SDK).
- **Platform notes**: `gaid`/`idfa` only returned when Adjust, AppsFlyer, or Meta App Events (iOS-only) plugin is configured; otherwise omitted. IDFA requires ATT permission — until granted it returns the zeroed placeholder `00000000-0000-0000-0000-000000000000`. GAID omitted on devices without Google Play Services (e.g. Huawei), is user-resettable, and increasingly OS-restricted.
- **Gotchas**: `median_device_info()` must exist at page load. Always compare `idfa` against the zeroed placeholder before using for attribution. NPM library helper: `getPlatform()` returns `web`, `ios`, or `android`.
- **Minimal snippet**:
  ```javascript
  function median_device_info(deviceInfo) { console.log(deviceInfo); }
  median.run.deviceInfo();
  var deviceInfo = await median.deviceInfo();
  ```
- **Source**: https://docs.median.co/docs/device-info

## median.clipboard.set / median.clipboard.get (Clipboard)
- **Purpose**: Write text to or read the device clipboard from the web app.
- **Call signatures**:
  - `median.clipboard.set({ data: "..." })`
  - `median.clipboard.get()` — promise, or `median.clipboard.get({ callback: "fnName" })` — named callback
- **Parameters**:
  - set: `data` | string | Required | The text string to write to the clipboard
  - get: `callback` | string (function name) | Optional | Named callback for legacy/native contexts
- **Response shape** (both get patterns):
  | Property | Type | Description |
  |---|---|---|
  | `data` | string | Clipboard text content. Present on success. |
  | `error` | string | Error message. Present when clipboard could not be read. |
- **Platform notes**: iOS 14+ may show an OS banner that the app accessed the clipboard (cannot be suppressed). Android 10+ may restrict clipboard reads when app not in foreground — trigger reads from explicit user action.
- **Minimal snippet**:
  ```javascript
  median.clipboard.set({ data: "PROMO2024" });
  median.clipboard.get().then(function (result) {
    if (result.data) { console.log("Clipboard contents:", result.data); }
    else { console.error("Clipboard error:", result.error); }
  });
  ```
- **Source**: https://docs.median.co/docs/clipboard

## median.share.sharePage (Prompt Share Dialogue)
- **Purpose**: Trigger the native iOS/Android share dialog to share the current URL (or a given URL) plus optional text.
- **Call signatures**:
  - `median.share.sharePage()`
  - `median.share.sharePage({url: 'https://median.co/about'})`
  - `median.share.sharePage({url: 'https://median.co/about', text: 'Visit Median here'})`
- **Parameters**: `url` (optional; defaults to current URL), `text` (optional; shared along with URL). Docs give no explicit types table — both are strings.
- **Callback/promise**: none documented.
- **Platform notes**: Callable from JS context including native tab menus/sidebars. Share options depend on installed apps; simulators may show limited options.
- **Minimal snippet**:
  ```javascript
  median.share.sharePage({url: 'https://median.co/about', text: 'Visit Median here'});
  ```
- **Source**: https://docs.median.co/docs/prompt-share-dialogue

## median_app_resumed() (App Resumed Callback)
- **Purpose**: Page-defined function automatically invoked when the app resumes from background — use for data refresh, content updates, or full reload.
- **Call signature**: you define `function median_app_resumed() {...}` on the page or via Custom JavaScript; native invokes it. No `median.*` call.
- **Parameters**: none.
- **Callback shape**: none (it IS the callback).
- **Platform notes**: On Android, `median_app_resumed()` is also triggered after any native permission prompt.
- **Gotchas**: Must be present on the current page. For kiosk/signage auto-reload, Median offers a separate native plugin (sales contact required) that bypasses JavaScript.
- **Minimal snippet**:
  ```javascript
  function median_app_resumed() { window.location.reload(); }
  ```
- **Source**: https://docs.median.co/docs/app-resumed-callback

## median.keyboard (Keyboard State Tracking)
- **Purpose**: Get/subscribe to on-screen keyboard visibility and size; toggle iOS keyboard accessory view.
- **Call signatures**:
  - `median.keyboard.info({'callback': function})` — current state; optional callback, otherwise returns a Promise
  - `median.keyboard.listen(function)` — subscribe to state changes; `median.keyboard.listen("")` stops listening
  - `median.keyboard.showAccessoryView(true|false)` — iOS only
- **Parameters**: `callback` (function, optional for `info`), listener function for `listen`, boolean for `showAccessoryView`.
- **Response shape** (both `info` and each `listen` event — verbatim):
  ```javascript
  {
    'visible': BOOL,
    'keyboardWindowSize': { 'width': INT, 'height': INT },
    'visibleWindowSize': { 'width': INT, 'height': INT }
  }
  ```
- **Platform notes**: Accessory view toggle is iOS-only (spell check/formatting toolbar above keyboard).
- **Gotchas**: `visibleWindowSize` = remaining visible area with keyboard shown — use it to reposition sticky footers/CTAs.
- **Minimal snippet**:
  ```javascript
  function hideKeyboard(data) {
    if (data.visible) { median.tabNavigation.setTabs({ enabled: false }); }
    else { median.tabNavigation.setTabs({ enabled: true }); }
  }
  median.keyboard.listen(hideKeyboard);
  median.keyboard.showAccessoryView(false); // iOS only
  ```
- **Source**: https://docs.median.co/docs/keyboard-state-tracking

## median.share.downloadFile / median.share.downloadImage (Download File)
- **Purpose**: Download (and optionally open) a file on the user's device, or save an image to the photo gallery.
- **Call signatures**:
  - `median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true|false})`
  - `median.share.downloadImage({url: 'https://yoursite.com/file.jpg'})`
- **Parameters**: `url` string (public URL required — localhost NOT supported); `open` boolean — **Android-only parameter**.
- **Callback/promise**: none documented.
- **Platform notes**:
  - iOS: file downloads then is passed to the native "Open with" system dialog.
  - Android with "Downloads Folder Enabled" (Permissions tab): downloads silently; `open: true` shows "Open with" dialog.
  - Android with "Private to App" enabled (Permissions tab): always shows "Open with" dialog after download.
- **Gotchas**: URLs must be publicly available. Alternative hands-off path: set `Content-Disposition: inline` (view in webview) vs `attachment` (download) headers on your server. PDFs view in-app via Apple PDFKit (iOS) / Pdf-Viewer library (Android); both viewers include a Print icon (iOS bottom-right, Android top-right).
- **Minimal snippet**:
  ```javascript
  median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true});
  median.share.downloadImage({url: 'https://yoursite.com/file.jpg'});
  ```
- **Source**: https://docs.median.co/docs/download-file

## median.webview.clearCache (Clear Webview Cache)
- **Purpose**: Programmatically clear the webview cache to force fresh asset loads (dev/troubleshooting).
- **Call signature**: `median.webview.clearCache()`
- **Parameters**: none.
- **Android-only appConfig.json options** (under `general`):
  ```json
  "general": { "androidClearCache": true }
  "general": { "androidCacheMode": "no_cache" }  // no_cache | cache_only | cache_else_network | default
  ```
  | Mode | Description |
  |---|---|
  | `no_cache` | Always load from the network, never use the cache. |
  | `cache_only` | Only use cached content, never hit the network. |
  | `cache_else_network` | Use cache if available, otherwise fetch from the network. |
  | `default` | Use the system's default caching behavior. |
- **Gotchas**: Don't call on every app start — kills perf by re-downloading assets. Production fix is asset versioning (e.g. `application_782374982.js`) + correct cache headers. Docs don't state whether cookies/localStorage/sessions are affected — test login behavior before production use.
- **Minimal snippet**: `median.webview.clearCache();`
- **Source**: https://docs.median.co/docs/clear-webview-cache

## median.screen.setBrightness (Screen Brightness)
- **Purpose**: Set screen brightness programmatically (0–100% via 0–1.0), optionally restoring on navigation.
- **Call signatures** (verbatim from docs):
  - `median.screen.setBrightness({'brightness':'0.8'})`
  - `median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true})`
  - `median.screen.setBrightness({'brightness':'default'})`
- **Parameters**: `brightness` string — `'0'` to `'1.0'` (e.g. `'0.8'` = 80%) or `'default'`; `restoreOnNavigation` boolean — reverts to previous setting after page navigation.
- **Platform notes**: none stated.
- **Gotchas**: Docs snippets contain a stray trailing `'` after the closing paren (e.g. `...})';`) — a doc typo; the call itself ends `})`. Pass values as strings per docs examples.
- **Minimal snippet**:
  ```javascript
  median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true});
  median.screen.setBrightness({'brightness':'default'});
  ```
- **Source**: https://docs.median.co/docs/screen-brightness

## median.screen.keepScreenOn / keepScreenNormal (Keep Screen On)
- **Purpose**: Prevent the screen from sleeping at runtime (wake lock).
- **Call signatures**:
  - `median.screen.keepScreenOn()` — enable
  - `median.screen.keepScreenNormal()` — disable
- **Parameters**: none.
- **Platform notes**: Default mode can be set in app configuration on the **Interface** tab.
- **Minimal snippet**:
  ```javascript
  median.screen.keepScreenOn();
  median.screen.keepScreenNormal();
  ```
- **Source**: https://docs.median.co/docs/keep-screen-on

## median.android.screen.fullScreen / normal (Full Screen)
- **Purpose**: Hide Android navigation/status bars (or iOS landscape sidebars) for a full-screen experience.
- **Call signatures**:
  - `median.android.screen.fullScreen()` — enable
  - `median.android.screen.normal()` — disable
- **Parameters**: none.
- **Platform notes**: JS command is Android-scoped. iOS full screen in landscape (hiding sidebars) is configured via the **Interface** tab, not the bridge. Default mode also settable on Interface tab for Android.
- **Gotchas** (docs warning): With Full Screen enabled on Android, the keyboard **overlays** web content and can break forms. Fixes: (1) disable full screen via bridge on form pages, or (2) use the keyboard listener functions to toggle full screen when the keyboard shows/hides.
- **Minimal snippet**:
  ```javascript
  median.android.screen.fullScreen();
  median.android.screen.normal();
  ```
- **Source**: https://docs.median.co/docs/full-screen

## Screen Orientation (config-only, no bridge command)
- **Purpose**: Lock or auto-rotate app orientation per OS/device type.
- **Call signature**: none — App configuration only (App Studio setting). Modes: **Auto-Rotation**, **Fixed Portrait**, **Fixed Landscape**; customizable by Operating System (iOS/Android) and Device Type (Phone/Tablet).
- **Parameters**: n/a.
- **Platform notes / Gotchas**: Enabling Fixed Portrait on iPads **automatically disables multi-tasking capabilities**.
- **Source**: https://docs.median.co/docs/screen-orientation

## median.screen.setColorScheme / resetColorScheme (Dark Mode)
- **Purpose**: Force Light/Dark/Auto color scheme for native UI (nav/tab/sidebar menus) and drive web content via `prefers-color-scheme`.
- **Call signatures**:
  - `median.screen.setColorScheme("light")`
  - `median.screen.setColorScheme("dark")`
  - `median.screen.setColorScheme("auto")` — follow device mode
  - `median.screen.resetColorScheme()` — revert to app-config default
- **Parameters**: single string argument `"light" | "dark" | "auto"` (not an object, per docs examples).
- **Callback/promise**: none documented.
- **Platform notes**: Web content dark mode: app sets `prefers-color-scheme` to `light`/`dark` dynamically with device mode. Median also sets a `data-color-scheme-option` property on the page with current app setting (`light`, `dark`, or `auto`) for building a toggle.
- **Gotchas**: Docs wrap calls in `if (navigator.userAgent.indexOf("median") > -1)` to guard against running outside the app. Recommend CSS variables for light/dark palettes.
- **Minimal snippet**:
  ```javascript
  if (navigator.userAgent.indexOf("median") > -1) {
    median.screen.setColorScheme("dark");
    median.screen.resetColorScheme();
  }
  ```
- **Source**: https://docs.median.co/docs/dark-mode

## median.haptics.trigger + median_device_shake (Haptics) — plugin required
- **Purpose**: Trigger haptic vibration effects (impact/notification feedback) and respond to device shake gesture.
- **Prerequisite**: Haptics plugin enabled under Native Plugins (App Studio). Physical device required — no meaningful feedback in simulators.
- **Call signatures**:
  - `median.haptics.trigger({ style: styleName })`
  - `function median_device_shake() {...}` — page-defined; invoked when user shakes the device (also insertable via Custom JavaScript)
- **Parameters**: `style` string:
  - iOS + Android: `impactLight`, `impactMedium`, `impactHeavy`, `notificationSuccess`, `notificationWarning`, `notificationError`
  - Android only: `tick`, `click`, `double_click`
- **Callback/promise**: none documented for trigger.
- **Gotchas**: Android-only styles have no effect on iOS. Function name must match exactly: `median_device_shake`.
- **Minimal snippet**:
  ```javascript
  median.haptics.trigger({ style: 'impactMedium' });
  function median_device_shake(){
    document.querySelector('.sideNavigation').style.visibility = "visible";
  };
  ```
- **Source**: https://docs.median.co/docs/haptics

## Calendar (plugin required — .ics link interception, no bridge call)
- **Purpose**: Let users add events to their device calendar by tapping `.ics` or `data:text/calendar` links.
- **Prerequisite**: Calendar plugin enabled (plug-and-play) + JS Bridge enabled.
- **Call signature**: none — plugin intercepts link taps: (1) intercepts navigation, (2) downloads the `.ics`, (3) parses it, (4) prompts user to confirm adding to native calendar.
- **Two implementation paths**:
  1. Hosted `.ics` file: `<a href="https://yoursite.com/event.ics">Add to Calendar</a>`
  2. Data URI (verbatim docs example):
  ```html
  <a href="data:text/calendar;charset=utf8,BEGIN:VCALENDAR
  VERSION:2.0
  BEGIN:VEVENT
  DTSTART:20240117T190000Z
  DTEND:20240117T200000Z
  SUMMARY:Doctor Appointment
  DESCRIPTION:Annual checkup
  LOCATION:123 Main St
  END:VEVENT
  END:VCALENDAR">Add to Calendar</a>
  ```
- **Gotchas**: Malformed iCalendar data → missing/incorrect event details; validate required fields `DTSTART`, `DTEND`, `SUMMARY`. Not a standalone web calendar — only intercepts `.ics`.
- **Source**: https://docs.median.co/docs/calendar

## median.contacts.* (Native Contacts) — plugin required
- **Purpose**: Sync/pick device contacts for form completion, CRM entry, lookups by email/phone.
- **Prerequisite**: Native Contacts plugin enabled. App auto-prompts for permission on first access.
- **Call signatures** (each accepts a callback or returns a Promise):
  - `median.contacts.getPermissionStatus({ callback: myCallback })` / `await median.contacts.getPermissionStatus({})` → `{ "status": "STRING" }`
  - `median.contacts.getAll({ callback: myCallback })` / `await median.contacts.getAll({})`
  - `median.contacts.pickContact({ callback: myCallback, multiple: BOOL })` / `await median.contacts.pickContact({ multiple: false })` — native picker UI
- **Parameters**: `callback` function (for callback style); `multiple` boolean — `true` = multi-select, `false` = single contact only (pickContact).
- **Permission status values**:
  - iOS: `granted`, `denied`, `restricted` (admin prohibited, e.g. parental controls/MDM), `notDetermined`
  - Android: `granted`, `denied` (not yet asked OR explicitly denied)
- **Response shape**: `{ success: true, contacts: [...] }` when permission granted.
- **Contact fields**: iOS — `birthday`, `namePrefix`, `givenName`, `middleName`, `familyName`, `previousFamilyName`, `nameSuffix`, `nickname`, `phoneticGivenName`, `phoneticMiddleName`, `phoneticFamilyName`, `organizationName`, `departmentName`, `jobTitle`, `note` (pickContact only), `phoneNumbers`[], `emailAddresses`[], `postalAddresses`[]. Android — `birthday`, `givenName`, `familyName`, `companyName`, `companyTitle`, `note`, `phoneNumbers`[], `emailAddresses`[], `postalAddresses`[].
- **Array field schemas** (verbatim):
  - phoneNumbers: `{ "label": "STRING", "phoneNumber": "STRING" }`
  - emailAddresses: `{ "label": "STRING", "emailAddress": "STRING" }`
  - postalAddresses: `{ "label", "street", "city", "state" (iOS only), "region" (Android only), "postalCode", "country", "isoCountryCode" (iOS only), "subAdministrativeArea" (iOS only), "subLocality" (iOS only) }` — all strings
- **Gotchas**: iOS `note` field not returned by `getAll()` by default — needs `com.apple.developer.contacts.notes` entitlement (Apple approval). `note` IS available via `pickContact()` on iOS without extra entitlement. `pickContact` is the recommended approach for explicit user selection.
- **Source**: https://docs.median.co/docs/native-contacts

## median.storage.app / median.storage.cloud (Native Datastore) — plugin required
- **Purpose**: Persist key-value data on-device (App Storage) or synced to the user's cloud account (Cloud Storage).
- **Backends**: App Storage = Android SharedPreferences / iOS UserDefaults. Cloud Storage = Android SharedPreferences + Android Backup Service / iOS Apple Keychain Services.
- **Call signatures** (identical shape for `app` and `cloud`):
  - `median.storage.app.set({ key: KEY, value: VALUE, statuscallback: statcb })`
  - `median.storage.app.get({ key: KEY, callback: cbRead })` or promise: `median.storage.app.get({ key: KEY }).then(...)`
  - `median.storage.app.delete({ key: KEY, statuscallback: statcb })`
  - `median.storage.app.deleteAll({ statuscallback: statcb })`
  - same four under `median.storage.cloud.*`
- **Parameters** (set): `key` String Required; `value` String Required; `statuscallback` Function No — receives `{ status }`. (get): `key` String Required; `callback` Function Required for callback style — receives `result.data` and `result.status`.
- **Response shape**: get callback/promise → `{ data: <stored value>, status: STRING }`; set/delete/deleteAll statuscallback → `{ status: STRING }`.
- **Status values**: `success`, `read-error`, `write-error`, `delete-error`, `preference-not-found`.
- **Storage limits**: Android App Storage no limit / Cloud up to 5 MB; iOS App Storage up to 500 KB / Cloud up to 16 MB.
- **Platform notes / Gotchas**: Android cloud sync via BackupManager is asynchronous and system-scheduled — no guaranteed sync time. iOS Keychain sync depends on user's iCloud Keychain settings. Keys are case-sensitive. Android file locations: `DATA/data/APP_PACKAGE_NAME/shared_prefs/user_preferences.xml` (app) and `user_preferences_backup.xml` (cloud). Cloud storage survives reinstall (iOS Keychain is hardware-backed; suitable for sensitive values).
- **Minimal snippet**:
  ```javascript
  median.storage.app.set({ key: 'theme', value: 'dark', statuscallback: statcb });
  median.storage.app.get({ key: 'theme' }).then(function(result) {
    console.log(result.data); console.log(result.status);
  });
  median.storage.cloud.deleteAll({ statuscallback: statcb });
  ```
- **Source**: https://docs.median.co/docs/native-datastore

## median.readerModal (Reader Modal) — iOS, Apple entitlement required
- **Purpose**: Open an external-link account-management/payments modal for Reader apps (magazines, books, audio, video, etc.) per Apple's Reader app policy.
- **Prerequisite**: Apple **External Link Account entitlement** permission (special approval from Apple).
- **Call signatures**:
  - `const { canMakePayments } = await median.readerModal.canMakePayments();`
  - `median.readerModal.showModal();`
- **Parameters**: none.
- **Response shape**: `canMakePayments()` resolves `{ canMakePayments: BOOL }` (per destructuring in docs).
- **Platform notes**: Modal appears on iOS 16+ devices. Always verify payment ability before showing the modal.
- **Minimal snippet**:
  ```javascript
  const { canMakePayments } = await median.readerModal.canMakePayments();
  if (canMakePayments) { median.readerModal.showModal(); }
  ```
- **Source**: https://docs.median.co/docs/reader-modal

## median.modal.launch (Secure Modal) — plugin required
- **Purpose**: Open a secure iOS WKWebView window with external scripting blocked (required by e.g. Apple Pay JS API).
- **Call signature**:
  - `median.modal.launch({ 'url': 'https://applepaydemo.apple.com', 'autoClosePath': '/payment-complete', 'callback': modal_closed })` — callback method
  - same object without `callback`, chained `.then(function (data) {...})` — promise method
- **Parameters**: `url` string (required in examples); `autoClosePath` string optional — auto-closes modal when that URL path loads (e.g. after successful payment); `callback` function optional.
- **Response shape** (returned when modal closes — verbatim):
  ```javascript
  {
    closeMethod: "closeButton" | "autoClosePath",
    url: STRING,            // URL with complete path shown when closed
    params: {               // Params on the URL when closed e.g. ?status=success results in status: success
      key: value,
      key2: value
    }
  }
  ```
- **Platform notes / Gotchas**: On iOS 16+, Apple permits Apple Pay directly in an app webview — optionally check iOS version via `deviceInfo` and skip the modal there.
- **Minimal snippet**:
  ```javascript
  median.modal.launch({
     'url': 'https://applepaydemo.apple.com',
     'autoClosePath':'/payment-complete',
     'callback': modal_closed
  });
  function modal_closed(data) {
    if (data.closeMethod == 'closeButton') { alert('modal closed via button at: ' + data.url); }
    else if (data.closeMethod == 'autoClosePath') { alert('modal closed automatically'); }
  }
  ```
- **Source**: https://docs.median.co/docs/secure-modal

## median.appreview.prompt (App Review) — plugin required
- **Purpose**: Prompt the user to rate/review the app on the Apple App Store or Google Play.
- **Prerequisite**: App Review plugin installed. iOS config: set your App Store ID (numerical `idXXXXXXX` from the listing URL; e.g. YouTube = `544007664`).
- **Call signature**: `median.appreview.prompt({ callback: appReviewComplete })`
- **Parameters**: `callback` — optional function run once the review modal has been closed.
- **Platform notes / Gotchas**:
  - iOS: Apple controls whether the prompt actually displays (anti-spam); dev builds always show it; **no effect in TestFlight builds**.
  - Android: Google Play enforces a time-based rate-limit quota on review prompts. Best practices: avoid direct CTA buttons, target natural completion points, fall back to redirecting to the Play Store listing when quota likely exceeded.
- **Minimal snippet**:
  ```javascript
  median.appreview.prompt({ callback: appReviewComplete });
  ```
- **Source**: https://docs.median.co/docs/app-review

## median_share_to_app(data) (Share into App) — plugin required
- **Purpose**: Native share extension — users share URLs from any app into yours; app launches and invokes your JS callback with URL + subject.
- **Prerequisite**: Share into App plugin enabled + **rebuild required** after enabling. iOS: configure an **App Group** identifier (enter the bare iOS Bundle ID, e.g. `com.example.app` — platform derives `group.<bundleId>`); the App Group MUST be manually registered in Apple Developer Portal and enabled on **both** the app App ID and the `<bundleId>.ShareExtension` App ID, or release builds fail. Also set the URL scheme protocol under **Link Handling** in App Studio.
- **Call signature**: page-defined callback invoked by native:
  ```javascript
  function median_share_to_app(data) {
    alert(data.url);     // Shared URL
    alert(data.subject); // Page title or shared text
  }
  ```
- **Callback shape**: `data.url` (shared URL), `data.subject` (page title / shared text). Not all share sources supply a subject.
- **Gotchas**: Function must be defined on the website before the native callback fires (JS Bridge enabled). Plugin config changes require a new binary.
- **Source**: https://docs.median.co/docs/share-into-app

## median.webScreenshot.* (Web Screenshot) — plugin required
- **Purpose**: Capture and share visible web content or a specific element as an image, or export it as a Blob.
- **Call signatures**:
  - `median.webScreenshot.shareScreen({ url: "...", text: "..." })` — capture + native-share visible webview area (excludes native navigation bars)
  - `median.webScreenshot.shareElement(element, { url: "...", text: "..." })` — capture + share a specific element (e.g. a `<div>`)
  - `median.webScreenshot.captureScreen()` — returns screenshot as a `Blob`
  - `median.webScreenshot.captureElement(element)` — returns element screenshot as a `Blob`
- **Parameters**: `element` (DOM element, for the Element variants); `url` and `text` strings (optional context shared alongside the image).
- **Callback/promise**: share variants use the native share dialog; capture variants return a Blob synchronously per docs examples.
- **Gotchas**: `shareScreen` captures only the visible portion, excluding native UI (native navigation bars). Blob output can be uploaded to your server or fed to another plugin (docs cite Social Share).
- **Minimal snippet**:
  ```javascript
  median.webScreenshot.shareScreen({ url: "https://www.median.dev", text: "Median Developer Demo" });
  const element = document.getElementById("content");
  median.webScreenshot.shareElement(element, { url: "https://median.dev/", text: "Median Developer Demo" });
  const screenShotblob = median.webScreenshot.captureScreen();
  const elementBlob = median.webScreenshot.captureElement(element);
  ```
- **Source**: https://docs.median.co/docs/web-screenshot

---

## Coverage

**Fetched OK (21/21):**
1. device-info
2. clipboard
3. prompt-share-dialogue
4. app-resumed-callback
5. keyboard-state-tracking
6. download-file
7. clear-webview-cache
8. screen-brightness
9. keep-screen-on
10. full-screen
11. screen-orientation
12. dark-mode
13. haptics
14. calendar
15. native-contacts
16. native-datastore
17. reader-modal
18. secure-modal
19. app-review
20. share-into-app
21. web-screenshot

**FETCH FAILED:** none. (Transient Firecrawl rate limits were retried successfully after waiting; every page ultimately retrieved.)

**Notes for skill authors:**
- Plugin-required APIs (must be enabled in App Studio → Native Plugins): haptics, calendar, native-contacts, native-datastore, secure-modal (implied), app-review, share-into-app, web-screenshot, reader-modal (Apple entitlement).
- Auto-invoked page callbacks (define at load time): `median_device_info()`, `median_app_resumed()`, `median_device_shake()`, `median_share_to_app(data)`.
- No bridge command exists for: screen orientation (config-only) and calendar (`.ics` link interception).
- Recurring async pattern: pass `{ callback: fn }` or omit for a Promise (clipboard.get, keyboard.info, contacts.*, storage get).
