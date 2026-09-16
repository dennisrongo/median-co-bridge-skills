---
name: median-device-apis
description: "Read device/app info and enforce minimum app versions; drive clipboard, share sheet, downloads, keyboard state, app-resume, webview cache and zoom, geolocation without double prompts, and Android camera capture settings. Use when web code needs device data, location, camera, keyboard, or webview controls inside a Median app."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/device-info
---

# Median Device APIs

Read real-time device/app state and drive device-level hardware from your web app: device info (platform, identifiers, ad IDs) with minimum-app-version enforcement, clipboard read/write, the native share sheet, file downloads, geolocation without double permission prompts, camera/file-upload behavior, webview cache clearing and zoom, app-resume detection, and on-screen keyboard state. All of these are core JavaScript Bridge APIs — no native plugins required; two configuration gates apply (Location Services permission for Android geolocation, Viewport Width = "WebView Scale" for zoom — see Prerequisites).

Everything here runs inside a Median-built iOS/Android app wrapping your existing website. In a desktop browser the `median` object does not exist; guard calls accordingly.

## When to Use

Reach for this skill when the task involves:

- Reading platform, OS version, app version, `installationId`, carrier, locale, or ad identifiers (`idfa`/`gaid`) for analytics, attribution, or support flows
- Copying text to / reading text from the device clipboard (promo codes, referral links)
- Opening the native iOS/Android share dialog for the current page or a specific URL
- Downloading files (PDFs, images) to the device, optionally showing the "Open with" dialog
- Clearing the webview cache programmatically during development or troubleshooting
- Getting the user's location via `navigator.geolocation` without the iOS double-permission prompt, and requesting Android location permission at runtime
- Tuning Android camera capture (quality preset, gallery saving) around `<input type="file">` upload flows
- Reading or changing webview zoom at runtime (e.g. a per-user display-density setting)
- Enforcing a minimum app version and redirecting outdated builds to an update page
- Refreshing data (or forcing a reload) when the user returns the app to the foreground
- Tracking on-screen keyboard visibility/size to reposition sticky footers or CTAs, or toggling the iOS keyboard accessory view

Don't use for:

- Push notifications or auth — use the `median-push-auth` skill
- Analytics SDKs, attribution, or in-app purchases — use the `median-analytics-iap` skill
- Screen brightness, keep-awake, fullscreen, orientation, dark mode, swipe gestures — use `median-screen-controls`
- Plugin-gated features (haptics, contacts, datastore, review prompts, modals, screenshots) — use `median-native-features`
- QR/barcode/document scanning, NFC — use `median-scanning`

## Prerequisites

- **JavaScript Bridge enabled** in App Studio (required for every bridge call). No additional native plugins are required for any API in this skill.

Per-feature gates beyond the bridge:

| Capability | Gate |
|---|---|
| Geolocation (Android) | **Location Services permission enabled** in app configuration — required before `promptLocationServices()` can grant anything |
| Webview zoom | **Interface tab → Viewport Width = "WebView Scale"** — `getZoom`/`setZoom` depend on this setting |
| Camera capture settings | None — but `setCaptureQuality`/`saveToGallery` are **Android-only** |
| Device info, clipboard, share, downloads, keyboard, app-resume, clearCache | None |

- Field-level extras: `idfa`/`gaid` are only returned when an ad/attribution SDK plugin (Adjust, AppsFlyer, or Meta App Events — iOS-only) is configured. `carrierNames` on Android requires the `READ_PHONE_STATE` permission.
- Download behavior on Android depends on the **Permissions tab** setting: "Downloads Folder Enabled" vs "Private to App".

## Quick Reference

| Task | Exact call |
|---|---|
| Receive device info automatically after every page load | define `function median_device_info(deviceInfo) {...}` at page load |
| Fetch device info on demand | `var deviceInfo = await median.deviceInfo();` |
| Trigger the auto-callback manually | `median.run.deviceInfo();` |
| Enforce a minimum app version | compare `deviceInfo.appVersion` in `median_device_info()`; redirect with `window.location.replace(...)` — snippet in [references/device-info.md](references/device-info.md) |
| Write to clipboard | `median.clipboard.set({ data: "PROMO2024" });` |
| Read clipboard (promise) | `median.clipboard.get().then(...)` |
| Read clipboard (named callback) | `median.clipboard.get({ callback: "fnName" })` |
| Open native share dialog | `median.share.sharePage({url: '...', text: '...'});` |
| Download a file (open = Android-only param) | `median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true});` |
| Save an image to photo gallery | `median.share.downloadImage({url: 'https://yoursite.com/file.jpg'});` |
| iOS: request geolocation once ready | define `function median_geolocation_ready() {...}` — native calls it when location services initialize |
| Android/web: call geolocation immediately | `if (!navigator.userAgent.includes('MedianIOS')) median_geolocation_ready();` |
| Android: prompt location permission | `median.android.geoLocation.promptLocationServices();` — no-op if already granted |
| Android: check location services | `median.android.geoLocation.isLocationServicesEnabled({'callback': function});` → `{ "enabled": true \| false }` |
| Android: set camera capture quality | `median.camera.setCaptureQuality("low");` — `"high"` (default) \| `"low"` |
| Android: stop saving captures to gallery | `median.camera.saveToGallery(false);` — re-enable with `true` |
| Clear webview cache | `median.webview.clearCache();` |
| Get webview zoom level | `median.webview.getZoom();` |
| Set webview zoom to 110% | `median.webview.setZoom(1.1);` |
| React when app resumes from background | define `function median_app_resumed() {...}` on the page |
| Current keyboard state | `median.keyboard.info({'callback': function})` or its Promise |
| Subscribe to keyboard show/hide | `median.keyboard.listen(function)` |
| Stop subscribing | `median.keyboard.listen("")` |
| Toggle iOS keyboard accessory view | `median.keyboard.showAccessoryView(true|false)` |

## How It Works

### Call forms: library, NPM, protocol

- **Library (default):** `median.module.function()` — injected into the page inside the app.
- **NPM:** `import Median from "median-js-bridge";` then the same call shape with a capitalized `Median` (e.g. `Median.clipboard.set(...)`). Enable *Website Overrides > JavaScript Frameworks and NPM* in App Studio and disable library injection.
- **Protocol:** `median://<module>/<function>` via an anchor `href` or `window.location.href`, with parameters `encodeURIComponent()`-encoded. For callback-style commands the documented form is `median://run/<command>?callback=<globalFunctionName>`; the callback resolves against the parent window.

See the `median-bridge-setup` skill for bridge readiness (`median_library_ready()`), app detection, and SPA guidance.

### Device Info — `median.deviceInfo()`

Three interchangeable patterns (all documented):

```javascript
function median_device_info(deviceInfo) { console.log(deviceInfo); } // auto-invoked after every page load
median.run.deviceInfo();                                              // manual trigger of your callback
var deviceInfo = await median.deviceInfo();                           // promise form
```

Full response payload (platform, appId, appVersion, installationId, idfa/gaid, isFirstLaunch, …) and the complete field-by-field table are in [references/device-info.md](references/device-info.md). Key gotchas:

- `median_device_info()` **must exist at page-load time** — it cannot be added asynchronously or deferred.
- `idfa` returns the zeroed placeholder `00000000-0000-0000-0000-000000000000` until ATT permission is granted. Always compare against that placeholder before using it for attribution.
- NPM helper: `Median.getPlatform()` resolves to `'web'`, `'ios'`, or `'android'`.

### Clipboard — `median.clipboard.set` / `median.clipboard.get`

```javascript
median.clipboard.set({ data: "PROMO2024" });
median.clipboard.get().then(function (result) {
  if (result.data) { console.log("Clipboard contents:", result.data); }
  else { console.error("Clipboard error:", result.error); }
});
```

`get` also accepts a named callback: `median.clipboard.get({ callback: "fnName" })`. Both patterns resolve `{ data }` on success or `{ error }` on failure. iOS 14+ may show an unsuppressable OS banner on clipboard access; Android 10+ may block reads when the app is not foregrounded — trigger reads from explicit user actions.

### Share sheet — `median.share.sharePage()`

```javascript
median.share.sharePage();                                                    // shares current URL
median.share.sharePage({url: 'https://median.co/about', text: 'Visit Median here'});
```

`url` defaults to the current URL; `text` is shared alongside it. Callable from JS context including native tab menus/sidebars.

### Downloads — `median.share.downloadFile` / `downloadImage`

```javascript
median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true});
median.share.downloadImage({url: 'https://yoursite.com/file.jpg'});
```

`url` must be **publicly reachable** (localhost is not supported). `open` is an **Android-only parameter**. Behavior matrix (iOS "Open with" dialog, Android downloads-folder vs private-to-app modes, in-app PDF viewers, `Content-Disposition` alternative) is in [references/clipboard-share-downloads.md](references/clipboard-share-downloads.md).

### Geolocation — `median_geolocation_ready()` + `median.android.geoLocation.*`

On iOS, Median's native layer shims `navigator.geolocation` so the web API uses the native implementation. You must delay geolocation calls until native location services are initialized, or the user gets a **double permission prompt** — once from the native app, once from the webview. Native calls `median_geolocation_ready()` on your page when it is safe to proceed:

```javascript
// Automatically called by Median when iOS native location services are ready
function median_geolocation_ready() {
  navigator.geolocation.getCurrentPosition(locationSuccess, locationError, locationOptions);
}

// Fallback: Call immediately on non-iOS platforms
if (!navigator.userAgent.includes('MedianIOS')) {
  median_geolocation_ready();
}
```

On Android, geolocation permission must be requested at runtime through the bridge (and the **Location Services permission** must be enabled in your app configuration first):

```javascript
median.android.geoLocation.promptLocationServices(); // native dialog; no-op if already granted

median.android.geoLocation.isLocationServicesEnabled({'callback': function});
// Return value: { "enabled": true | false }
```

Full parameter/response details and the demo app link are in [references/location-camera.md](references/location-camera.md). Continuous background location is a separate native plugin — see the [Background Location docs](https://docs.median.co/docs/background-location).

### Camera & file upload — `median.camera.setCaptureQuality` / `saveToGallery`

File and image upload works out of the box on both platforms via standard `<input type="file">` — the WebView presents the system file/camera picker, no plugin or configuration required. Android-only extras via the bridge:

```javascript
median.camera.setCaptureQuality("low");  // "high" (default) | "low" (reduced resolution/file size)
median.camera.saveToGallery(false);      // stop saving captures to the device gallery (persists across sessions)
```

WebRTC (`navigator.mediaDevices.getUserMedia()`) camera/microphone permission prompts are also handled natively on both platforms — no extra configuration, but the page must be served over HTTPS. Details in [references/location-camera.md](references/location-camera.md).

### App resumed — `median_app_resumed()`

There is no `median.*` call — you define the callback and native invokes it when the app returns from background:

```javascript
function median_app_resumed() { window.location.reload(); }
```

On Android it also fires after any native permission prompt. With the NPM package you can instead register `Median.appResumed.addListener(() => {...})`.

### Keyboard state — `median.keyboard.*`

```javascript
function hideKeyboard(data) {
  if (data.visible) { median.tabNavigation.setTabs({ enabled: false }); }
  else { median.tabNavigation.setTabs({ enabled: true }); }
}
median.keyboard.listen(hideKeyboard);
median.keyboard.showAccessoryView(false); // iOS only
```

Every `info()` result and `listen()` event carries `{ visible, keyboardWindowSize, visibleWindowSize }`. `visibleWindowSize` is the remaining visible area with the keyboard shown — use it to reposition sticky footers/CTAs. Full details in [references/keyboard-resume-cache.md](references/keyboard-resume-cache.md).

### Webview cache & zoom — `median.webview.clearCache` / `getZoom` / `setZoom`

```javascript
median.webview.clearCache();
median.webview.getZoom();
median.webview.setZoom(1.1); // 110%
```

`clearCache` takes no parameters and has no callback. Android-only `appConfig.json` options (`androidClearCache`, `androidCacheMode` with its four modes) are documented in [references/keyboard-resume-cache.md](references/keyboard-resume-cache.md). Do not call `clearCache` on every app start — it forces re-download of all assets.

Zoom requires the App Studio setting **Interface → Viewport Width = "WebView Scale"** — without it the zoom calls have nothing to scale. `setZoom(1.1)` sets 110%; the initial zoom level can also be configured in App Studio. Full details and the demo link are in [references/webview-zoom.md](references/webview-zoom.md).

## Pitfalls

- **Auto-invoked callbacks must exist at page load.** `median_device_info()` and `median_app_resumed()` are called by native code; if they are defined late (SPA async mount, deferred script), the invocation is missed. For SPAs, expose them as globals (`window.median_device_info = ...`) or use NPM listeners.
- **IDFA placeholder trap.** Treat `00000000-0000-0000-0000-000000000000` as "no ATT grant", not as a real identifier.
- **Promise vs callback inconsistency.** `clipboard.get`, `keyboard.info`, and `deviceInfo` support promise or callback styles; `sharePage`, `downloadFile`, `downloadImage`, `clearCache`, `clipboard.set` document no return value. Don't `await` the latter group.
- **`open` is Android-only.** Passing `open` on iOS has no documented effect — iOS always hands the file to the native "Open with" flow after download.
- **Public URLs only for downloads.** Localhost and internal hosts fail silently on device.
- **Android 10+ clipboard restriction.** Background reads can be blocked; read the clipboard only from a user gesture.
- **Don't clearCache on launch.** Production cache-busting is asset versioning + correct HTTP cache headers; the docs warn clearCache kills performance by re-downloading assets, and its effect on cookies/localStorage/sessions is undocumented — test login flows before shipping.
- **iOS geolocation double prompt.** Calling `navigator.geolocation` before native fires `median_geolocation_ready()` prompts twice (native app + webview). Use the wait pattern; call immediately only on non-iOS (`!navigator.userAgent.includes('MedianIOS')`).
- **Android location needs config + runtime prompt.** `promptLocationServices()` shows the native dialog only if permission is not already granted, and requires the Location Services permission enabled in app configuration. The documented check form is `isLocationServicesEnabled({'callback': function})` returning `{ "enabled": true | false }` — not a promise.
- **Camera toggles are Android-only and sticky.** `setCaptureQuality`/`saveToGallery` are not supported on iOS, and `saveToGallery(false)` persists across sessions — re-enable with `true` deliberately.
- **Zoom calls require "WebView Scale".** Set Interface → Viewport Width = "WebView Scale", or `getZoom`/`setZoom` have nothing to scale.
- **`median.webview.reload()` is not documented.** Community snippets show a `reload()` bridge command, but no current official docs page documents it (checked Webview Zoom, Clear Webview Cache, NPM Basic Usage, Refresh Button). Reload via `window.location.reload()`, the native top-nav Refresh Button, or Pull-to-Refresh (both App Studio config) instead.
- **Outside the app there is no `median` object.** Guard with `navigator.userAgent.indexOf("median") > -1` or `Median.isNativeApp()` (NPM) so the same code runs on the mobile web.

## Verification

1. Device info: `await median.deviceInfo()` returns `platform`, `appVersion`, `installationId` inside the app; outside the app the guard no-ops.
2. Version enforcement: temporarily set `MIN_APP_VERSION` above the installed `appVersion` — the app redirects to your update-required page, and Back does not return to gated content (`location.replace` removes it from history).
3. Geolocation (iOS): fresh install → the first `getCurrentPosition` triggers exactly ONE permission prompt (via `median_geolocation_ready()`).
4. Geolocation (Android): `promptLocationServices()` shows the native dialog once; a second call is a no-op; the `isLocationServicesEnabled` callback receives `{ "enabled": true }` after granting.
5. Camera: `<input type="file">` opens the system picker on both platforms; on Android, `setCaptureQuality("low")` yields visibly smaller captures and `saveToGallery(false)` keeps new captures out of the gallery — re-check after an app restart, since the setting persists.
6. Zoom: with Viewport Width = "WebView Scale", `setZoom(1.1)` renders content at 110% and `getZoom()` reflects the change; without the setting, note the no-op before shipping.
7. Clipboard: `set` then `get` round-trips the string (expect the unsuppressable iOS 14+ banner on read).
8. Resume: background the app and foreground it — `median_app_resumed()` fires (on Android it also fires after native permission prompts).

## References

- [references/device-info.md](references/device-info.md) — full `deviceInfo` response JSON, field-by-field table, IDFA/GAID/ATT notes, minimum-app-version enforcement snippet
- [references/clipboard-share-downloads.md](references/clipboard-share-downloads.md) — clipboard get/set, share sheet, downloadFile/downloadImage platform behavior
- [references/keyboard-resume-cache.md](references/keyboard-resume-cache.md) — keyboard state tracking, app-resumed callback, clearCache + Android cache modes
- [references/location-camera.md](references/location-camera.md) — `median_geolocation_ready()` wait pattern, Android runtime location permission, camera capture quality/gallery toggles, WebRTC notes
- [references/webview-zoom.md](references/webview-zoom.md) — WebView Scale requirement, `getZoom`/`setZoom`, initial-zoom configuration, reload caveat
- [Repo recipes](../../recipes.md) — version gate + push identity on login; keyboard-aware forms on Android
- Official docs: [Device Info](https://docs.median.co/docs/device-info) · [Clipboard](https://docs.median.co/docs/clipboard) · [Share Dialog](https://docs.median.co/docs/prompt-share-dialogue) · [Download File](https://docs.median.co/docs/download-file) · [App Resumed](https://docs.median.co/docs/app-resumed-callback) · [Keyboard State](https://docs.median.co/docs/keyboard-state-tracking) · [Clear Webview Cache](https://docs.median.co/docs/clear-webview-cache) · [Location Services](https://docs.median.co/docs/location-services) · [Camera and File Uploads](https://docs.median.co/docs/camera-and-file-uploads) · [WebRTC Audio & Video](https://docs.median.co/docs/web-rtc) · [Webview Zoom](https://docs.median.co/docs/webview-zoom)
- Live demo pages: [Device Info](https://median.dev/device-info) · [Location Services](https://median.dev/location-services/) · [Camera](https://median.dev/camera/) · [Android Camera Settings](https://median.dev/camera-settings) · [Keyboard](https://median.dev/keyboard/) · [Webview Zoom](https://median.dev/webview-zoom/)
