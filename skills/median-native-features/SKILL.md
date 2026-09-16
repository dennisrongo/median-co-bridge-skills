---
name: median-native-features
description: "Play background audio with lock-screen controls via the background media player, stream native video through JW Player, trigger haptics, add calendar events, read contacts, persist key-value data on-device or to the cloud, present reader/secure modals, prompt app reviews, receive shares, capture web screenshots. Use when a Median app needs plugin-gated native features or background/lock-screen audio."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/native-plugins-overview
---

# Median Native Features

Plugin-gated bridge features that reach deeper into the device: haptic feedback and shake detection, calendar event adds, native contacts access, on-device/cloud key-value storage, reader and secure modals, the native app-review prompt, share-into-app, web screenshots, and background media playback with native lock-screen/notification controls. Every API in this skill requires the **JavaScript Bridge enabled** *plus* its specific **Native Plugin** toggled on in App Studio — calls silently do nothing (or never fire) when the plugin is off.

Everything runs inside a Median-built iOS/Android app wrapping your existing website. In a desktop browser the `median` object does not exist; guard calls accordingly.

## When to Use

Reach for this skill when the task involves:

- Triggering haptic vibration (impact/notification styles, Android tick/click) or reacting to a device shake
- Letting users add events to their device calendar (`.ics` link interception — no bridge command)
- Reading device contacts (single pick or full list) for form completion, CRM entry, lookups
- Persisting key-value data on-device (App Storage) or synced to the user's cloud account (Cloud Storage)
- Opening Apple's Reader-app external-link modal, or a secure WKWebView modal (e.g. for Apple Pay JS)
- Prompting the user to rate the app on the App Store / Google Play
- Accepting shares from other apps into yours (`median_share_to_app` callback)
- Capturing/sharing a screenshot of the page or a specific element
- Playing audio in the background with lock-screen/notification controls (single track or playlist, metadata, seek)
- Native full-screen video playback through the JW Player plugin

Don't use for:

- Clipboard/share-sheet/downloads/device-info/keyboard — `median-device-apis`
- Brightness/keep-awake/fullscreen/orientation/dark mode — `median-screen-controls`
- Push notifications, biometrics, login — `median-push-auth`
- QR/barcode/document/NFC scanning — `median-scanning`

## Prerequisites

- **JavaScript Bridge enabled** in App Studio (required for every bridge call).
- **Native Plugins** — each feature below needs its plugin enabled in App Studio → Native Plugins, and the app rebuilt:

| Feature | Plugin / gate |
|---|---|
| Haptics | Haptics plugin |
| Calendar | Calendar plugin |
| Native Contacts | Native Contacts plugin (auto-prompts for permission on first access) |
| Native Datastore | Native Datastore plugin |
| Reader Modal | Apple **External Link Account entitlement** (special approval from Apple); iOS 16+ |
| Secure Modal | Secure Modal plugin |
| App Review | App Review plugin; iOS requires your App Store ID (numerical `idXXXXXXX`) |
| Share into App | Share into App plugin + rebuild; iOS also needs an App Group (see pitfalls) |
| Web Screenshot | Web Screenshot plugin |
| Background Media | Native Media Player plugin (released Mar 2026) |
| JW Player (video) | JW Player plugin (released Apr 2026) + valid Android/iOS license keys; optional Google Cast App ID for iOS Chromecast |

- Haptics requires a **physical device** — no meaningful feedback in simulators.
- `idfa`/`gaid`-style attribution SDKs are not part of this skill (see `median-analytics-iap`).

## Quick Reference

| Task | Exact call |
|---|---|
| Trigger a haptic | `median.haptics.trigger({ style: 'impactMedium' });` |
| React to device shake | define `function median_device_shake() {...}` on the page |
| Add event to calendar | no bridge command — link to a hosted `.ics` or `data:text/calendar` URI; plugin intercepts |
| Check contacts permission | `await median.contacts.getPermissionStatus({})` → `{ "status": "STRING" }` |
| Pick one contact via native UI | `median.contacts.pickContact({ multiple: false })` |
| Get all contacts | `median.contacts.getAll({})` |
| Store a value on device | `median.storage.app.set({ key: 'theme', value: 'dark', statuscallback: statcb });` |
| Read a stored value | `median.storage.app.get({ key: 'theme' }).then(...)` |
| Delete one key / all keys | `median.storage.app.delete({ key: KEY, statuscallback: statcb })` / `median.storage.app.deleteAll({ statuscallback: statcb })` |
| Same for cloud-synced storage | identical calls under `median.storage.cloud.*` |
| Check Reader-modal eligibility | `const { canMakePayments } = await median.readerModal.canMakePayments();` |
| Show Reader modal | `median.readerModal.showModal();` |
| Launch secure modal | `median.modal.launch({ 'url': '...', 'autoClosePath': '/payment-complete', 'callback': modal_closed })` |
| Prompt for app review | `median.appreview.prompt({ callback: appReviewComplete });` |
| Receive shares from other apps | define `function median_share_to_app(data) {...}` on the page |
| Share a page screenshot | `median.webScreenshot.shareScreen({ url: "...", text: "..." });` |
| Share one element | `median.webScreenshot.shareElement(element, { url: "...", text: "..." });` |
| Capture as Blob | `median.webScreenshot.captureScreen()` / `captureElement(element)` |
| Play audio in the background | `const { success } = await median.backgroundMedia.playTrack({ url, title, artist, album, time: 50 });` |
| Stream a playlist | `median.backgroundMedia.streamPlaylist({ currentTrackNumber: 0, tracks: [{ url, title }] });` |
| Pause / stop playback | `median.backgroundMedia.pause();` / `median.backgroundMedia.stop();` |
| Get player status | `await median.backgroundMedia.getPlayerStatus()` → `{ currentTime (ms), isPaused, album, artist, title, artwork, url }` |
| Play video via JW Player | `await median.jwplayer.play({ file: 'https://example.com/video.mp4', startTime: 30 });` |

## How It Works

### Call forms: library, NPM, protocol

- **Library (default):** `median.module.function()` — injected into the page inside the app.
- **NPM:** `import Median from "median-js-bridge";` then the same call shape with a capitalized `Median` (e.g. `Median.haptics.trigger(...)`). Enable *Website Overrides > JavaScript Frameworks and NPM* in App Studio and disable library injection. NPM also offers listeners for auto-callbacks: `Median.deviceShake.addListener(fn)` and `Median.shareToApp.addListener(fn)`.
- **Protocol:** `median://<module>/<function>` via an anchor `href` or `window.location.href`, parameters `encodeURIComponent()`-encoded.

See the `median-bridge-setup` skill for bridge readiness (`median_library_ready()`) and app detection.

### Haptics — `median.haptics.trigger` + `median_device_shake()`

```javascript
median.haptics.trigger({ style: 'impactMedium' });
function median_device_shake(){
  document.querySelector('.sideNavigation').style.visibility = "visible";
};
```

Cross-platform styles: `impactLight`, `impactMedium`, `impactHeavy`, `notificationSuccess`, `notificationWarning`, `notificationError`. Android-only: `tick`, `click`, `double_click` (no effect on iOS). The shake callback name must match `median_device_shake` exactly and be defined at page load (or inserted via App Studio Custom JavaScript).

### Calendar — `.ics` interception, NO bridge command

There is **no `median.*` call for calendar**. The Calendar plugin intercepts link taps: navigation is intercepted → the `.ics` is downloaded → parsed → the user is prompted to add it to their native calendar. Two implementation paths:

```html
<!-- 1. Hosted .ics file -->
<a href="https://yoursite.com/event.ics">Add to Calendar</a>

<!-- 2. Data URI -->
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

Validate required iCalendar fields (`DTSTART`, `DTEND`, `SUMMARY`) — malformed data produces missing/incorrect event details. Full details in [references/haptics-calendar.md](references/haptics-calendar.md).

### Native contacts — `median.contacts.*`

```javascript
median.contacts.getPermissionStatus({ callback: myCallback });   // or await ...({})
median.contacts.pickContact({ callback: myCallback, multiple: false });
median.contacts.getAll({ callback: myCallback });
```

Each call accepts `{ callback: fn }` or returns a Promise. Permission statuses differ by platform: iOS has `granted` / `denied` / `restricted` / `notDetermined`; Android has only `granted` / `denied` (denied covers both not-yet-asked and explicitly denied). Responses are `{ success: true, contacts: [...] }`.

**`pickContact` vs `getAll` note-field entitlement:** on iOS, the `note` field is **not returned by `getAll()`** by default — it needs the `com.apple.developer.contacts.notes` entitlement (requires Apple approval). `note` **IS available via `pickContact()`** without the extra entitlement, and `pickContact` is the recommended approach for explicit user selection. Full field schemas (iOS vs Android, phone/email/postal arrays) in [references/contacts-datastore.md](references/contacts-datastore.md).

### Native datastore — `median.storage.app` / `median.storage.cloud`

```javascript
median.storage.app.set({ key: 'theme', value: 'dark', statuscallback: statcb });
median.storage.app.get({ key: 'theme' }).then(function(result) {
  console.log(result.data); console.log(result.status);
});
median.storage.cloud.deleteAll({ statuscallback: statcb });
```

Identical call shape for `app` (on-device) and `cloud` (user's cloud account). Reads are async — `get` takes a `callback` or returns a Promise resolving `{ data, status }`; writes/deletes report via `statuscallback` receiving `{ status }` (`success`, `read-error`, `write-error`, `delete-error`, `preference-not-found`).

**Platform notes:** Android cloud sync (BackupManager) is asynchronous and system-scheduled — no guaranteed sync time. iOS cloud sync rides Apple Keychain (hardware-backed, survives reinstall, suitable for sensitive values). Keys are case-sensitive. Storage limits: Android app no limit / cloud 5 MB; iOS app 500 KB / cloud 16 MB. App Storage is available offline — it is the storage leg of the cross-skill offline scan → store → sync recipe ([Repo recipes](../../recipes.md)).

### Reader modal — `median.readerModal` (iOS, Apple entitlement)

```javascript
const { canMakePayments } = await median.readerModal.canMakePayments();
if (canMakePayments) { median.readerModal.showModal(); }
```

For Reader apps (magazines, books, audio, video) needing an external-link account/payments modal per Apple's Reader app policy. Requires Apple's **External Link Account entitlement** (special approval). Appears on iOS 16+; always verify `canMakePayments` before showing the modal.

### Secure modal — `median.modal.launch`

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

Opens a secure iOS WKWebView with external scripting blocked (required by e.g. the Apple Pay JS API). Omit `callback` and chain `.then(...)` for the promise form. When the modal closes you receive `{ closeMethod, url, params }`. On iOS 16+ Apple permits Apple Pay directly in an app webview — optionally check the OS version via `deviceInfo` and skip the modal.

### App review — `median.appreview.prompt`

```javascript
median.appreview.prompt({ callback: appReviewComplete });
```

iOS: Apple controls whether the prompt actually displays (anti-spam); dev builds always show it; **no effect in TestFlight builds** — set your App Store ID in plugin config. Android: Google Play enforces a time-based rate-limit quota; target natural completion points, avoid direct "Rate us" CTAs, and fall back to redirecting to the Play Store listing.

### Share into app — `median_share_to_app(data)`

```javascript
function median_share_to_app(data) {
  alert(data.url);     // Shared URL
  alert(data.subject); // Page title or shared text
}
```

With the Share into App plugin enabled, users can share URLs from any app into yours; the app launches and invokes this page-defined callback. Define it **before** the native callback fires (page-load time). Not all share sources supply a `subject`.

**iOS setup:** configure an **App Group** identifier (enter the bare iOS Bundle ID, e.g. `com.example.app` — the platform derives `group.<bundleId>`); the App Group MUST be manually registered in the Apple Developer Portal and enabled on **both** the app App ID and the `<bundleId>.ShareExtension` App ID, or release builds fail. Also set the URL scheme protocol under **Link Handling** in App Studio. **Enabling the plugin requires a rebuild** — plugin config changes need a new binary.

### Web screenshot — `median.webScreenshot.*`

```javascript
median.webScreenshot.shareScreen({ url: "https://www.median.dev", text: "Median Developer Demo" });
const element = document.getElementById("content");
median.webScreenshot.shareElement(element, { url: "https://median.dev/", text: "Median Developer Demo" });
const screenShotblob = median.webScreenshot.captureScreen();
const elementBlob = median.webScreenshot.captureElement(element);
```

`shareScreen` captures the visible webview area (excluding native navigation bars) and opens the native share dialog; `shareElement` targets a specific DOM element. The `capture*` variants return a `Blob` you can upload to your server or feed to another plugin.

### Background media player — `median.backgroundMedia.*`

```javascript
// Single track — metadata fields are optional (used when the audio file has none embedded)
var track = {
  "url": "https://Your_Audio_URL",
  "title": "Example Title",
  "album": "Example Album",
  "artist": "Example Artist",
  "time": 50   // optional: start playing at 50 seconds
};
const { success } = await median.backgroundMedia.playTrack(track);

// Playlist — auto-advances when a track ends; Next/Previous in the lock-screen UI
var playlist = {
  "currentTrackNumber": 0,
  "tracks": [
    { "url": "https://Your_Audio_URL_1", "title": "Example 1 Title" },
    { "url": "https://Your_Audio_URL_2" }
  ]
};
median.backgroundMedia.streamPlaylist(playlist);

median.backgroundMedia.pause();
median.backgroundMedia.stop();   // Android: stops playback + removes the notification; iOS: pauses only

const status = await median.backgroundMedia.getPlayerStatus();
// → { currentTime (ms), isPaused, album, artist, title, artwork, url }
```

- **Seek:** `median.backgroundMedia.playTrack(50)` seeks the *currently playing* media to 50 s *(docs show a bare number argument — odd form, preserved verbatim)*; `"time": 50` inside the track object starts a track at 50 s.
- **Metadata:** manually supplied metadata overrides what's embedded in the audio file; with none available, defaults are Title `Playing Media`, Album/Artist = app name, and the app icon substitutes for missing artwork (artwork can only be pulled from the audio URL itself).
- **Background Mode (web-player sync):** to keep an in-app web player in sync across background/resume, define global `getPlayerStatus()` (native calls it when the app is backgrounded — return the web player's status) and `updatePlayerStatus(mediaPlayerStatus)` (called on resume). Note `currentTime` is **milliseconds** native-side but seconds on HTML `<audio>`/`<video>` elements.
- *(Naming quirk: the Mar 2026 release notes title this the "Background Audio plugin" and link it to `docs.median.co/docs/background-audio` — a page about microphone recording. The `median.backgroundMedia.*` playback API documented above lives on the [Native Media Player](https://docs.median.co/docs/native-media-player) page.)*

### JW Player — `median.jwplayer.*` (native video, short pointer)

Released Apr 2026, the JW Player plugin wraps JW Player's native iOS/Android SDKs for full-screen native video playback — a single `file`, a `files` array (playlist), or a JW `playlistUrl`, plus `startTime`. It auto-initializes from the license keys entered in App Studio, or manually:

```javascript
const result = await median.jwplayer.initialize({ licenseKey: 'YOUR_JW_PLAYER_LICENSE_KEY' });
await median.jwplayer.play({ file: 'https://example.com/video.mp4', startTime: 30 });
```

The callback fires when the player is configured on iOS (`{ success }`) but only on close on Android (with `position`, `state`, `file`, ...). Chromecast on iOS initializes automatically at launch when a Google Cast App ID is configured. Full parameter tables: [JW Player docs](https://docs.median.co/docs/jwplayer).

## Pitfalls

- **Plugin-gated: enable before coding.** Every API here silently fails without its Native Plugin enabled in App Studio and a rebuilt binary. Share into App plugin changes always require a new binary.
- **Auto-invoked callbacks must exist at page load.** `median_device_shake()` and `median_share_to_app(data)` are invoked by native code; defining them late (SPA async mount) misses the call. Expose as globals (`window.median_device_shake = ...`) or use NPM listeners (`Median.deviceShake`, `Median.shareToApp`).
- **Calendar and orientation-style surprises:** there is NO calendar bridge command — only `.ics`/`data:text/calendar` link interception. Don't invent `median.calendar.*` calls.
- **iOS contacts `note` entitlement.** `getAll()` omits `note` without the `com.apple.developer.contacts.notes` entitlement; `pickContact()` returns it without extra approval. Prefer `pickContact` for user-driven selection.
- **Datastore is async everywhere.** `get` is promise/callback (`{ data, status }`); `set`/`delete`/`deleteAll` report through `statuscallback` — they don't resolve the stored value. Android cloud sync is system-scheduled with no guaranteed timing; don't treat cloud writes as immediately durable.
- **Review prompts are rate-limited by the OS.** iOS may not show the prompt at all (and never in TestFlight); Google Play enforces a quota. Provide a Play Store listing redirect as a fallback.
- **Reader modal requires Apple's special entitlement** — without the External Link Account approval, `showModal()` is not usable. Always gate on `canMakePayments()`.
- **Secure modal `params` come from the closing URL** — e.g. `?status=success` yields `{ status: 'success' }`. Parse them to detect payment outcome, not just `closeMethod`.
- **Haptics need a real device** — simulators give no feedback; Android-only styles (`tick`, `click`, `double_click`) do nothing on iOS.
- **Web screenshot captures visible content only** — `shareScreen` excludes native navigation bars and off-screen content; capture an element instead when you need all of it.
- **Background media `stop()` differs per platform** — Android stops playback and removes the notification; iOS only pauses. Don't assume `stop()` clears the iOS lock-screen player.
- **Background-mode sync functions must be global** — `getPlayerStatus()` and `updatePlayerStatus(status)` are invoked by native code on background/resume (like `median_device_shake`); define them at page load. `currentTime` is milliseconds native-side but seconds on HTML media elements — convert before assigning.
- **JW Player calls fail without license keys** — `initialize()`/`play()` return `NOT_INITIALIZED` when the plugin isn't configured with valid Android/iOS license keys and the app rebuilt.
- **Outside the app there is no `median` object.** Guard with `navigator.userAgent.indexOf("median") > -1` or `Median.isNativeApp()` (NPM).

## Verification

1. Every API used has its Native Plugin enabled in App Studio and the app rebuilt — the feature's demo page (median.dev/...) behaves identically inside the app.
2. Haptics: `median.haptics.trigger({ style: 'impactMedium' })` vibrates a physical device; shaking the device fires `median_device_shake()`.
3. Calendar: tapping a hosted `.ics` link shows the native add-to-calendar prompt with correct `DTSTART`/`DTEND`/`SUMMARY`.
4. Contacts: first access prompts for permission; `pickContact({ multiple: false })` returns the chosen contact with `phoneNumbers[]`/`emailAddresses[]` populated.
5. Datastore: `set` → force-quit → relaunch → `get` returns the value and `statuscallback` reported `success`; keys match case exactly.
6. Reader modal: `canMakePayments()` is checked before `showModal()`; the modal appears on iOS 16+ with the entitlement approved.
7. Secure modal: closes via the close button AND via `autoClosePath`; `params` reflect the closing URL's query string.
8. App review: the prompt displays in a dev build (iOS) at a natural completion point; Android honors its rate-limit quota.
9. Share into app: sharing a URL from the OS share sheet launches the app and `median_share_to_app(data)` receives `data.url`.
10. Web screenshot: `shareScreen` opens the native share dialog showing the captured page; `captureElement(el)` returns a usable `Blob`.
11. Background media: play a track, press Home — playback continues, the lock screen/notification shows title/artist with Play/Pause, a playlist auto-advances, and `getPlayerStatus()` reflects `isPaused`/`currentTime`.
12. JW Player: `median.jwplayer.play({ file })` opens the full-screen native player; the iOS callback fires on configuration, Android's on close with playback state.

## References

- [references/haptics-calendar.md](references/haptics-calendar.md) — haptic styles, shake callback, calendar `.ics` interception paths
- [references/contacts-datastore.md](references/contacts-datastore.md) — contacts permission statuses, contact field schemas, storage limits/status values
- [references/modals-review.md](references/modals-review.md) — reader modal, secure modal response shape, app review platform behavior
- [references/share-screenshot.md](references/share-screenshot.md) — share-into-app setup (App Group, Link Handling), web screenshot calls
- [Repo recipes](../../recipes.md) — cross-skill recipes; the offline scan → store → sync recipe uses this skill's App Storage
- Official docs: [Haptics](https://docs.median.co/docs/haptics) · [Calendar](https://docs.median.co/docs/calendar) · [Native Contacts](https://docs.median.co/docs/native-contacts) · [Native Datastore](https://docs.median.co/docs/native-datastore) · [Reader Modal](https://docs.median.co/docs/reader-modal) · [Secure Modal](https://docs.median.co/docs/secure-modal) · [App Review](https://docs.median.co/docs/app-review) · [Share into App](https://docs.median.co/docs/share-into-app) · [Web Screenshot](https://docs.median.co/docs/web-screenshot) · [Native Media Player](https://docs.median.co/docs/native-media-player) · [JW Player](https://docs.median.co/docs/jwplayer)
- Live demo pages (open inside your app): https://median.dev/native-datastore/ · https://median.dev/native-media-player/ · https://median.dev/native-media-player-switch/ · https://median.dev/jwplayer
