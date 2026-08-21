---
name: median-device-apis
description: "Read device info; use clipboard, share, downloads, cache."
version: 0.1.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
---

# Median Device APIs

Read real-time device/app state and drive device-level hardware from your web app: device info (platform, identifiers, ad IDs), clipboard read/write, the native share sheet, file downloads, webview cache clearing, app-resume detection, and on-screen keyboard state. All of these are core JavaScript Bridge APIs — no extra native plugin toggles beyond the bridge itself.

Everything here runs inside a Median-built iOS/Android app wrapping your existing website. In a desktop browser the `median` object does not exist; guard calls accordingly.

## When to Use

Reach for this skill when the task involves:

- Reading platform, OS version, app version, `installationId`, carrier, locale, or ad identifiers (`idfa`/`gaid`) for analytics, attribution, or support flows
- Copying text to / reading text from the device clipboard (promo codes, referral links)
- Opening the native iOS/Android share dialog for the current page or a specific URL
- Downloading files (PDFs, images) to the device, optionally showing the "Open with" dialog
- Clearing the webview cache programmatically during development or troubleshooting
- Refreshing data (or forcing a reload) when the user returns the app to the foreground
- Tracking on-screen keyboard visibility/size to reposition sticky footers or CTAs, or toggling the iOS keyboard accessory view

Don't use for:

- Push notifications, auth, or analytics SDKs — use the `median-push-auth` skill
- Screen brightness, keep-awake, fullscreen, orientation, dark mode — use `median-screen-controls`
- Plugin-gated features (haptics, contacts, datastore, review prompts, modals, screenshots) — use `median-native-features`

## Prerequisites

- **JavaScript Bridge enabled** in App Studio (required for every bridge call).
- No additional native plugins are required for any API in this skill.
- Field-level extras: `idfa`/`gaid` are only returned when an ad/attribution SDK plugin (Adjust, AppsFlyer, or Meta App Events — iOS-only) is configured. `carrierNames` on Android requires the `READ_PHONE_STATE` permission.
- Download behavior on Android depends on the **Permissions tab** setting: "Downloads Folder Enabled" vs "Private to App".

## Quick Reference

| Task | Exact call |
|---|---|
| Receive device info automatically after every page load | define `function median_device_info(deviceInfo) {...}` at page load |
| Fetch device info on demand | `var deviceInfo = await median.deviceInfo();` |
| Trigger the auto-callback manually | `median.run.deviceInfo();` |
| Write to clipboard | `median.clipboard.set({ data: "PROMO2024" });` |
| Read clipboard (promise) | `median.clipboard.get().then(...)` |
| Read clipboard (named callback) | `median.clipboard.get({ callback: "fnName" })` |
| Open native share dialog | `median.share.sharePage({url: '...', text: '...'});` |
| Download a file (open = Android-only param) | `median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true});` |
| Save an image to photo gallery | `median.share.downloadImage({url: 'https://yoursite.com/file.jpg'});` |
| Clear webview cache | `median.webview.clearCache();` |
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

### Clear webview cache — `median.webview.clearCache()`

```javascript
median.webview.clearCache();
```

No parameters, no callback. Android-only `appConfig.json` options (`androidClearCache`, `androidCacheMode` with its four modes) are documented in [references/keyboard-resume-cache.md](references/keyboard-resume-cache.md). Do not call on every app start — it forces re-download of all assets.

## Pitfalls

- **Auto-invoked callbacks must exist at page load.** `median_device_info()` and `median_app_resumed()` are called by native code; if they are defined late (SPA async mount, deferred script), the invocation is missed. For SPAs, expose them as globals (`window.median_device_info = ...`) or use NPM listeners.
- **IDFA placeholder trap.** Treat `00000000-0000-0000-0000-000000000000` as "no ATT grant", not as a real identifier.
- **Promise vs callback inconsistency.** `clipboard.get`, `keyboard.info`, and `deviceInfo` support promise or callback styles; `sharePage`, `downloadFile`, `downloadImage`, `clearCache`, `clipboard.set` document no return value. Don't `await` the latter group.
- **`open` is Android-only.** Passing `open` on iOS has no documented effect — iOS always hands the file to the native "Open with" flow after download.
- **Public URLs only for downloads.** Localhost and internal hosts fail silently on device.
- **Android 10+ clipboard restriction.** Background reads can be blocked; read the clipboard only from a user gesture.
- **Don't clearCache on launch.** Production cache-busting is asset versioning + correct HTTP cache headers; the docs warn clearCache kills performance by re-downloading assets, and its effect on cookies/localStorage/sessions is undocumented — test login flows before shipping.
- **Outside the app there is no `median` object.** Guard with `navigator.userAgent.indexOf("median") > -1` or `Median.isNativeApp()` (NPM) so the same code runs on the mobile web.

## References

- [references/device-info.md](references/device-info.md) — full `deviceInfo` response JSON, field-by-field table, IDFA/GAID/ATT notes
- [references/clipboard-share-downloads.md](references/clipboard-share-downloads.md) — clipboard get/set, share sheet, downloadFile/downloadImage platform behavior
- [references/keyboard-resume-cache.md](references/keyboard-resume-cache.md) — keyboard state tracking, app-resumed callback, clearCache + Android cache modes
- Official docs: [Device Info](https://docs.median.co/docs/device-info) · [Clipboard](https://docs.median.co/docs/clipboard) · [Share Dialog](https://docs.median.co/docs/prompt-share-dialogue) · [Download File](https://docs.median.co/docs/download-file) · [App Resumed](https://docs.median.co/docs/app-resumed-callback) · [Keyboard State](https://docs.median.co/docs/keyboard-state-tracking) · [Clear Webview Cache](https://docs.median.co/docs/clear-webview-cache)
