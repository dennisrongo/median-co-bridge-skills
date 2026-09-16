# Median.co JavaScript Bridge — SETUP + NAVIGATION/UI Domain Research

Distilled from docs.median.co (fetched 2026-08-20). Verbatim signatures and parameters only; nothing invented.

---

# PART A: BRIDGE SETUP

## median (JavaScript Bridge Library) — overview
- Purpose: Injected runtime library that lets web pages control/configure the native app and access device hardware via `median.module.function()`.
- Two usage modes:
  1. **JavaScript Bridge Library** — injected into page DOM at runtime; available on any page displayed in the app. Call `median.module.function()` once ready.
  2. **Median Protocol** (`median://`) — direct, lower-level calls without the library. Use an HTML anchor `href` for simple commands or `window.location.href` for complex ones. Parameters with JSON/special characters must be `encodeURIComponent()`-encoded.
- Command chaining: use `median://nativebridge/multi` with an **array of URLs** instead of sequential calls (sequential calls can cause only the last command to run).
- Callback/promise: some bridge commands return JS promises — handle with `async/await` or `.then()/.catch()`:

```javascript
median.iap.purchase({ productID: 'product_id' })
  .then(function(data) { /* handle success */ })
  .catch(function(error) { /* handle error */ });
```

- Gotchas:
  - **GoNative → Median**: apps last updated on GoNative.io must use `gonative` not `median` (e.g. `gonative.statusbar.set()` or `gonative://statusbar/set`). Apps updated on Median.co may use either.
  - **`median_library_ready()` timing**: library initializes asynchronously. For page-load commands define `median_library_ready()` on the page; it is invoked after init. If the library initialized *before* your page defines the function, call it manually:

```javascript
function median_library_ready() {
  // Your code here
}
if (window.median) {
  window.median_library_ready();
}
```

  - `'median is not defined'` in console = normal in a desktop browser; the library only exists inside the app. Use the NPM package to develop/test outside the app (and disable library injection in App Studio → Website Overrides).
  - React/Vue/Angular: `median` may not be in scope. Options: (1) expose callbacks globally `window.callback_function = () => {}`; (2) use the NPM Package and toggle off library injection in App Studio → Website Overrides; (3) use the `median://` protocol.
  - `Median` not `median` when using NPM package — capitalized `Median` is the import; lowercase `median` is reserved for the injected library.
- Developer demo page (test in-app): https://median.dev/library-ready/
- Full docs index: https://docs.median.co/llms.txt (append `.md` to any docs URL for markdown)
- Source: https://docs.median.co/docs/javascript-bridge

## Detecting app usage (median vs browser)
- Purpose: Detect whether web content is running inside the Median native app so you can gate Bridge calls, adapt UI, or split analytics.
- Default user-agent strings appended to every HTTP request:

| Platform    | User agent string          | Legacy user agent string       |
| ----------- | -------------------------- | ------------------------------ |
| iOS App     | `MedianIOS/1.0 median`     | `GoNativeIOS/1.0 gonative`     |
| Android App | `MedianAndroid/1.0 median` | `GoNativeAndroid/1.0 gonative` |

- Frontend detection (verbatim):

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    // Running inside the Median app
    median.module.command({ 'parameter': 'value' });
    document.querySelector('.webNav').style.display = 'none';
    document.querySelector('.appOnly').style.display = 'block';
}
```

- Platform-specific check: `navigator.userAgent.indexOf('MedianIOS')` / `('MedianAndroid')` for ios/android distinction in analytics.
- Backend detection: read the `User-Agent` header (e.g. `req.header('User-Agent').indexOf('median') > -1` in Node/Express).
- UA string is configurable in App Studio → Website Overrides (see Custom User Agent); confirm with the Device-Info bridge function in the live app.
- Strategy comparison (from docs):

| Strategy                      | Best for                                   | Complexity |
| ----------------------------- | ------------------------------------------ | ---------- |
| User-agent detection (JS)     | Frontend UI changes, gating Bridge calls   | Low        |
| User-agent detection (server) | Serving different HTML, API responses      | Low        |
| Custom HTTP headers           | Clean server-side detection, backend APIs  | Low–Medium |
| Dedicated app URL             | Fully separate app vs. browser experiences | Medium     |

- Alternative strategies: (2) Custom HTTP headers configured in App Studio under Web Overrides → Custom Headers (e.g. `X-App-Platform: median-ios`); (3) dedicated app subdomain (e.g. `https://app.yoursite.com`) — every request is definitionally from the app.
- Best practice: wrap detection in a helper function (e.g. `logAnalyticsEvent(eventName, eventProperties)`) so app/browser branching stays centralized.
- Source: https://docs.median.co/docs/detecting-app-usage

## median-js-bridge (NPM package) — install & app config
- Purpose: Framework-agnostic NPM package for using the Bridge in SPAs (React, Angular, Vue.js, etc.).
- Install:
  - `npm install median-js-bridge --save`
  - `yarn add median-js-bridge`
  - Script tag: `<script type="text/javascript" src="https://unpkg.com/median-js-bridge@latest/dist/median.min.js"></script>`
- **Required app config**: enable *Website Overrides > JavaScript Frameworks and NPM* in App Studio. This prevents conflicts with the injected Bridge Library and activates required listener support. Omitting the package while enabling the option may result in unresponsive Bridge functions and callbacks.
- Gotchas: with `<script>` tag, avoid `@latest` in production — pin a version (e.g. `@1.12.3`) and upgrade deliberately; `@latest` gets every breaking change instantly.
- Links: https://github.com/gonativeio/median-javascript-bridge#readme · https://www.npmjs.com/package/median-js-bridge
- Source: https://docs.median.co/docs/npm-package

## Median (NPM basic usage): onReady / isNativeApp / getPlatform
- Purpose: Core entry points of the imported `median-js-bridge` package.
- **`Median` not `median`** when using the NPM package — capitalized `Median` object; lowercase `median` is reserved for the runtime-injected library. Configure IDE type hinting for completion.

```javascript
import Median from "median-js-bridge";
import React, { useEffect } from "react";

const App: React.FC = () => {
    useEffect(() => {
        Median.onReady(() => {
            window.alert("Median app ready!");
        });
    }, []);
}
```

| Function | Signature | Returns |
| --- | --- | --- |
| Ready hook | `Median.onReady(callback)` | calls `callback` once bridge is ready |
| App check | `Median.isNativeApp()` | `true` or `false` |
| Platform | `Median.getPlatform()` | promise resolving to `'web' \| 'android' \| 'ios'` |

- Promise-style example (OneSignal login + info, verbatim):

```javascript
useEffect(() => {
  Median.onReady(async () => {
    const result = await Median.onesignal.login('test-user');
    if (result?.success) {
      const info = await Median.onesignal.info();
      console.log(info?.oneSignalId);
    }
  })
}, []);
```

- Source: https://docs.median.co/docs/basic-usage

## Median listeners (NPM): addListener / removeListener
- Purpose: Register listener functions that respond to native events; events occurring *before* registration are queued and delivered once the listener initializes (reliable capture).
- Call signature: `const listenerId = Median.<interface>.addListener((data) => {...})` — each registration returns a `listenerId`.
- Removal: `Median.<interface>.removeListener(listenerId)`.
- Documented listener interfaces (verbatim examples):

| Listener | Example |
| --- | --- |
| App resumed | `Median.appResumed.addListener(() => { console.log("App resumed listener triggered"); })` |
| OneSignal push opened | `Median.oneSignalPushOpened.addListener((data) => { console.log(JSON.stringify(data)); })` |
| Share into app | `Median.shareToApp.addListener((data) => { console.log(data.url, data.subject); })` |
| Device shake (Haptics) | `Median.deviceShake.addListener(() => { console.log("Device shake listener"); })` |
| AppsFlyer conversion data | `Median.appsFlyerConversionData.addListener((conversionDataMap) => { console.log(conversionDataMap.af_status); })` |

- Best practices (docs): store the `listenerId` if you'll remove later; register early (in `useEffect`/lifecycle hook); remove listeners on unmount to avoid SPA memory leaks.
- Source: https://docs.median.co/docs/npm-package-usage-with-listeners

## Median.jsNavigation.url (SPA soft navigation)
- Purpose: Handle native-initiated navigation (tab bar taps, deep-link resume, push-notification resume) via the SPA router as soft loads instead of hard page loads.
- Call signature:

```javascript
Median.jsNavigation.url.addListener((url) => {
  // Use your SPA's routing logic to navigate to the URL
  // Soft load the requested content, triggering a hard page load only if necessary
});
```

- Triggered by: native navigation button clicks (e.g. bottom tab bar), deep-link app resume (email/SMS link launching app from background), push notification app resume (tapped notification with embedded URL).
- Gotchas (docs flow): without the listener, a warm-start deep link causes a costly full page load — or worse, **no load at all** if the app incorrectly determines the URL is already displayed from the initial cold-start load.
- Source: https://docs.median.co/docs/npm-package-spa-navigation

## Bridge callbacks inside an iframe
- Purpose: Proof-of-concept for accessing median callback data within an iframe.
- Pattern: iframe triggers the protocol command with a named callback; the **parent** page defines that global callback function and relays data into the iframe via `postMessage`.
- Parent setup (verbatim, abridged comments):

```html
<iframe src="IFRAME_URL" id="iframe"></iframe>
<script>
    function deviceInfoCallback(deviceInfo) {
        const iframe = document.getElementById('iframe');
        const message = { action: 'populateTextarea', deviceInfo: deviceInfo };
        iframe.contentWindow.postMessage(message, '*');
    }
</script>
```

- Iframe setup (verbatim):

```html
<button id="runDeviceInfo">Run Device Info</button>
<textarea id="deviceInfoOutput" placeholder="Device info will appear here..."></textarea>
<script>
    const button = document.getElementById('runDeviceInfo');
    button.addEventListener('click', () => {
        // Trigger the custom URL protocol
        window.location.href = 'median://run/median_device_info?callback=deviceInfoCallback';
    });
    window.addEventListener('message', (event) => {
        if (event.data.action === 'populateTextarea') {
            document.getElementById('deviceInfoOutput').value = JSON.stringify(event.data.deviceInfo);
        }
    });
</script>
```

- Protocol form: `median://run/<command>?callback=<globalFunctionName>` — callback resolves against the **parent** window.
- Gotchas: docs carry an explicit security disclaimer — the parent page and the callback function may be publicly accessible; review with a security team and follow best practices to prevent data leaks. (`postMessage(message, '*')` uses a wildcard target origin in the sample — tighten for production.)
- Source: https://docs.median.co/docs/iframe-callbacks

## Google Tag Manager template (bridge injection)
- Purpose: Median's GTM template connects a site to the JS Bridge by injecting `median.min.js` from unpkg in a sandboxed GTM environment, with error handling, debug logs, and `dataLayer` status pushes.
- Config fields (both optional):
  - `bridgeVersion` — version number of the bridge to inject; blank defaults to latest.
  - `userAgentValue` — name of a GTM user-defined variable holding the device UA string; included in console logs and the `dataLayer` payload (Browser/OS/Device type).
- Constructed CDN URL: `https://unpkg.com/median-js-bridge@[version]/dist/median.min.js`
- Required sandbox permissions: inject scripts from `https://unpkg.com/`; access global variables (push to `dataLayer`); log to console.
- dataLayer event pushed on completion — success payload:

```json
{ "event": "median_injected", "median_injected": "yes", "device_info": "[Value from userAgentValue field]" }
```

- Failure payload:

```json
{ "event": "median_injected", "median_injected": "no", "device_info": "[Value from userAgentValue field]" }
```

- Use the `median_injected` event to trigger subsequent tags.
- Source: https://docs.median.co/docs/google-tag-manager

---

# PART B: NATIVE NAVIGATION / UI

## Native navigation overview
- Purpose: Native navigation menus render instantly (before web content loads) and persist across internal and third-party pages — built at build time in App Studio or at runtime via the JS Bridge.
- Four component types: **Top Navigation Bar**, **Sidebar Navigation**, **Bottom Tab Bar**, and (iOS only) **Contextual Navigation Toolbar**.
- Benefits (docs): flexible UI control (build-time or runtime via Bridge), cross-service navigation via deep links + native tabs/sidebars, persistent navigation across internal/external pages, immediate rendering before any web content loads.
- Icon support: standard + custom libraries (Font Awesome, Material Design, custom SVG); separate active/inactive tab icons. See Custom Icons docs.
- Platform guidelines followed: iOS — Apple HIG; Android — Material Design.
- Source: https://docs.median.co/docs/native-navigation-overview

## Top Navigation Bar
- Purpose: Native header bar; visibility and content (titles, buttons) largely configured elsewhere.
- **Only visible if** using one of: Sidebar Navigation, Auto New Windows, Search, Refresh Button, or Custom Buttons.
- Set per-page title (verbatim):

```html
<script>
  function median_library_ready() {
    median.navigationTitles.setCurrent({ title: "Your Title" });
  }
</script>
```

- Developer demo: https://median.dev/top-navigation-bar/
- Source: https://docs.median.co/docs/top-navigation-bar

## Sidebar Navigation Menu
- Purpose: Slide-out sidebar menu (iOS + Android).
- The page is mostly screenshots; dynamic control is via `median.sidebar.setItems()` (see Dynamic Menu Items below).
- Developer demo: https://median.dev/sidebar-navigation/
- Source: https://docs.median.co/docs/sidebar-navigation-menu

## Bottom Tab Bar
- Purpose: Native bottom tab bar. iOS implementation follows Apple HIG (tab bars); Android follows Material Design (bottom navigation).
- Dynamic control via `median.tabNavigation.*` (see Dynamic Tab Menu / Selecting Tabs below).
- Developer demo: https://median.dev/tab-bar-navigation/
- Source: https://docs.median.co/docs/bottom-tab-bar

## median.tabNavigation.selectTab / deselect
- Purpose: Programmatically select or deselect bottom tab bar items.
- Call signatures (exact):
  - `median.tabNavigation.selectTab(1);` — tabs are **0-indexed**; `selectTab(1)` selects the *second* tab.
  - `median.tabNavigation.deselect();` — deselect all tabs.
- When needed: navigation from link/push/redirect onto a tab page; custom web navigation alongside the tab bar; highlighting the correct tab for dynamic routes, nested pages, or SPA views.
- URL-rule alternative: add a `regex` field per tab item in the tab-menu JSON; the tab shows active whenever a matching page is displayed. Docs example (note `"active": true` at root and `tabSelectionConfig` mapping):

```json
{
  "tabSelectionConfig": [ { "id": "1", "regex": ".*" } ],
  "tabMenus": [
    {
      "id": "1",
      "items": [
        {
          "subLinks": [],
          "label": "My Account",
          "icon": "fas fa-user",
          "url": "https://domain.com/account",
          "regex": "https://domain\\.domain/account.*"
        }
      ]
    }
  ],
  "active": true
}
```

- Best practices (docs): JS Bridge selection when the site controls nav state; URL rules when the active tab should follow the page URL; use specific regexes per tab to avoid conflicting active states; test on both iOS and Android; in SPAs, update tab selection on route change since no full page load occurs.
- Source: https://docs.median.co/docs/selecting-tabs

## median.sidebar.setItems (dynamic menu items)
- Purpose: Set sidebar navigation menu options at runtime.
- Call signature: `median.sidebar.setItems({"items":items,"enabled":true,"persist":true});`
- Parameters:
  - `enabled` — **required to activate the sidebar** if it is hidden.
  - `persist` — keeps the changes after the app is closed and reloaded.
- Item shape (from docs example): `label`, `url`, `icon` (optional, Font Awesome class e.g. `"fas fa-cog"`), `isGrouping: true` for a group header whose children live in `subLinks: [...]`, and `url` may be a `javascript:` URI.
- Verbatim example:

```javascript
var items = [{
    label: "Google",
    url: "https://google.com",
    icon: "fas fa-cog" // optional Font Awesome icon
}, {
    label: "Sample Grouping",
    isGrouping: true,
    subLinks: [{
        label: "Apple",
        url: "https://apple.com",
        icon: "fas fa-home" // optional
    }, {
        label: "Google",
        url: "https://google.com",
        icon: "fas fa-home" //optional
    }]
}, {
    label: "Sample Javascript",
    url: "javascript:alert('test')"
}];
median.sidebar.setItems({"items":items,"enabled":true, "persist":true});
```

- Source: https://docs.median.co/docs/dynamic-menu-items

## median.tabNavigation.setTabs (dynamic tab menu)
- Purpose: Define/replace bottom tab bar buttons at runtime, or toggle tab menu visibility. A default tab menu can be defined in app config and overwritten dynamically; config can be left blank with all tab menus set by the website.
- Call signatures:
  - Set/change tabs: `median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});`
  - Hide tab menu: `median.tabNavigation.setTabs({'enabled': false});`
  - Select/deselect: `median.tabNavigation.selectTab(1);` (0-indexed) / `median.tabNavigation.deselect();`
- Tab item shape: `icon` (optional, e.g. `"fas fa-cloud"`), `label`, `url` (can be `javascript:` URI).
- Verbatim example:

```javascript
var tabItems = [{
    "icon": "fas fa-cloud", //optional
    "label": "Tab 1",
    "url": "javascript:alert('You selected tab 1')"
}, {
    "icon": "fas fa-globe", //optional
    "label": "Tab 2",
    "url": "javascript:alert('You selected tab 2')"
}, {
    "icon": "fas fa-users", //optional
    "label": "Tab 3",
    "url": "javascript:alert('You selected tab 3')"
}];
median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});
```

- Source: https://docs.median.co/docs/dynamic-tab-menu

## median.navigationTitles.set / setCurrent (dynamic titles)
- Purpose: Configure Top Navigation Bar title text/images per URL — at build time (App Studio Dynamic Titles) or runtime (Bridge).
- Build-time config: list of `{regex, title}` or `{regex, showImage: true}` rules; rules prioritized top-to-bottom; no match → app name shown.
- Runtime — set full title structure (verbatim):

```javascript
var menuItems = {
    active: true,
    titles: [{
        showImage: true,
        regex: '/home'
    },{
        title: 'my title',
        regex: '.*'
    }]
}
median.navigationTitles.set({
  'persist':true,
  'data': menuItems});
```

  - `persist=true` saves titles for the next app launch; otherwise current-session only.
- Runtime — revert to build-time config:

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    median.navigationTitles.set({'persist':true});
}
```

- Runtime — one-off current page title (**URL-encode the title**):

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    median.navigationTitles.setCurrent({'title':'Hello%20World'})
}
```

```html
<a onclick="gonative.navigationTitles.setCurrent({'title':'Hello%20World'})">Set Current Page's Title</a>
```

- Note: the anchor example in the docs uses the `gonative.` prefix — both prefixes work on Median-updated apps.
- Source: https://docs.median.co/docs/dynamic-titles

## median.ios.contextualNavToolbar.set (iOS contextual navigation toolbar)
- Purpose: Native bottom toolbar with back (default), optional forward and refresh buttons — because iOS lacks a hardware back button.
- **iOS-only feature; NOT supported on Android.**
- Call signature (exact):

```javascript
// Show the contextual navigation toolbar, if conditions are met
median.ios.contextualNavToolbar.set({"enabled":true});
// Hide the contextual navigation toolbar
median.ios.contextualNavToolbar.set({"enabled":false});
```

- Display conditions when `enabled: true` — toolbar appears if ANY of: webview has back history; forward button enabled AND webview has forward history; refresh button enabled. `enabled: false` → always hidden.
- Build-time config via `toolbarNavigation` object in `appConfig.json`. Key options:
  - `visibilityByBackButton`: `"backButtonActive"` (default — show only when history exists) | `"always"`
  - `visibilityByPages`: `"allPages"` | `"specificPages"` (with `regexes` array; visibility based on the **first matched regex**)
  - `items[]`: each `{ "system": "back" | "refresh" | "forward", "enabled", "title", "titleType": "noText" | "defaultText" | "customText", "visibility": "allPages" | "specificPages", "urlRegex": [{ "enabled": true|false, "regex": ".*" }] }`
- Default config (verbatim):

```json
"toolbarNavigation": {
  "enabled": true,
  "visibilityByPages": "allPages",
  "visibilityByBackButton": "backButtonActive",
  "regexes": [ { "enabled": true, "regex": ".*" } ],
  "items": [
    { "system": "back", "titleType": "defaultText", "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] },
    { "system": "refresh", "enabled": false, "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] },
    { "system": "forward", "enabled": false, "titleType": "defaultText", "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] }
  ]
}
```

- Default appearance: Back button with text label (`< Back`); labels can be hidden or customized (`titleType: "customText"` + `title`).
- Source: https://docs.median.co/docs/contextual-navigation-toolbar

## median.statusbar.set (status bar styling)
- Purpose: Set status bar style/visibility/color at runtime; configure Light/Dark/Auto mode.
- Call signature (exact):

```javascript
// Dynamically
if (navigator.userAgent.indexOf('median') > -1) {
  median.statusbar.set({
    'style':'light',
    'color':'80ff0000',
    'overlay':true,
    'blur': true // optional - iOS only
  });
}

// Or on page load
function median_library_ready(){
  median.statusbar.set({
    'style':'light',
    'color':'80ff0000',
    'overlay':true,
    'blur': true // optional - iOS only
  });
}
```

- Parameters (verbatim from docs):

| Param | Type / values | Notes |
| --- | --- | --- |
| `style` | `'light'` \| `'dark'` \| `'auto'` | Text/icon colors. Light mode = black text, Dark mode = white text, Auto follows device Light/Dark mode setting. |
| `color` | `RRBBGG` or `AARRBBGG` hex | Solid status bar color. `'00000000'` = completely transparent. Example `'80ff0000'` = red at 50% alpha. |
| `overlay` | `true` \| `false` | `true` = web content extends underneath the status bar. `false` (default) = web content starts below the status bar. |
| `blur` | `true` \| `false` — **iOS only** | `true` applies a blur effect over the status bar color; `false` (default) keeps the color as specified by `color`. |

- Auto-match status bar to page background (built-in helper, called after library ready):

```javascript
function median_library_ready(){
  median_match_statusbar_to_body_background_color();
}
```

- Gotcha (Android): with `overlay: true` on Android, the **keyboard overlays web content**, breaking form completion. Docs remedies: (1) disable overlay via the Bridge on pages with forms; (2) use keyboard listener functions to toggle overlay when the keyboard is active/inactive.
- Developer demo: https://median.dev/status-bar/
- Source: https://docs.median.co/docs/status-bar

---

## Coverage

Fetched OK (18/18 — note `statusbar.md` in the task list resolves to slug `status-bar.md`):

| # | Page | Status |
| --- | --- | --- |
| 1 | /docs/javascript-bridge | OK (web_extract) |
| 2 | /docs/detecting-app-usage | OK (web_extract) |
| 3 | /docs/npm-package | OK (curl; web_extract rate-limited) |
| 4 | /docs/basic-usage | OK (curl) |
| 5 | /docs/npm-package-usage-with-listeners | OK (curl) |
| 6 | /docs/npm-package-spa-navigation | OK (web_extract) |
| 7 | /docs/iframe-callbacks | OK (web_extract) |
| 8 | /docs/google-tag-manager | OK (curl) |
| 9 | /docs/native-navigation-overview | OK (curl) |
| 10 | /docs/top-navigation-bar | OK (curl) |
| 11 | /docs/sidebar-navigation-menu | OK (curl; page is mostly screenshots — dynamic API covered by dynamic-menu-items) |
| 12 | /docs/bottom-tab-bar | OK (curl; page is mostly screenshots — dynamic API covered by dynamic-tab-menu/selecting-tabs) |
| 13 | /docs/selecting-tabs | OK (curl) |
| 14 | /docs/dynamic-menu-items | OK (curl) |
| 15 | /docs/dynamic-tab-menu | OK (curl) |
| 16 | /docs/dynamic-titles | OK (curl) |
| 17 | /docs/contextual-navigation-toolbar | OK (curl) |
| 18 | /docs/status-bar (task URL said `statusbar.md`) | OK (curl after discovering correct slug via llms.txt; `statusbar.md` returns an HTML 404 page) |

FETCH FAILED: none.
