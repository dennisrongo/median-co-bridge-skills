---
name: median-bridge-setup
description: "Wire the Median JavaScript Bridge into a web app — injected library, NPM package (`median-js-bridge`), or `median://` protocol; readiness gating (`median_library_ready()` / `Median.onReady()`), app-vs-browser detection, listeners, SPA navigation, iframe callbacks, GTM injection, and GoNative legacy (`gonative.`) migration. Use when bridge calls don't fire, `median` is undefined, or a GoNative.io-era app needs porting."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/javascript-bridge
---

# Median Bridge Setup

The Median.co JavaScript Bridge lets a web page control and configure the native iOS/Android app that displays it. There are three ways to wire it into an existing app: the **injected library** (`median.module.function()`), the **NPM package** (`median-js-bridge`, imported as capitalized `Median`), and the **`median://` protocol** (URL-based calls, no library). This skill covers choosing between them, the readiness/ordering rules, app-vs-browser detection, listeners, SPA navigation, iframe callbacks, and GTM injection.

## When to Use

Use this skill when:

- Wiring bridge calls into an existing web app that will run (or already runs) inside a Median-built app
- Choosing between the injected library, the NPM package, and the `median://` protocol
- Handling `median_library_ready()` timing or a "median is not defined" console error
- Supporting legacy GoNative apps (`gonative.` prefix instead of `median.`) — including migrating one using the full legacy command map in [references/gonative-legacy.md](references/gonative-legacy.md)
- Detecting whether the page is running in the app vs. a desktop/mobile browser (UA sniffing, custom headers, dedicated app URL)
- Registering/removing native-event listeners in an SPA
- Making native-initiated navigation (tab taps, deep links, push resumes) work with an SPA router
- Getting bridge callback data back into an iframe
- Injecting the bridge via Google Tag Manager

Don't use for:

- Calling specific native UI APIs (tab bar, sidebar, titles, status bar) — see the `median-navigation-ui` skill
- Device APIs, screen controls, push/auth, analytics/IAP — separate skills cover those

## Prerequisites

- A Median app (built on Median.co) at a plan tier that includes the features you invoke via the bridge.
- **NPM package route only**: enable *Website Overrides > JavaScript Frameworks and NPM* in App Studio. This prevents conflicts with the injected library and activates listener support. Leaving library injection on while also shipping the package can cause unresponsive bridge functions and callbacks.
- NPM install:

```bash
npm install median-js-bridge --save
# or
yarn add median-js-bridge
```

- Script-tag alternative (pin the version; don't ship `@latest`):

```html
<script type="text/javascript" src="https://unpkg.com/median-js-bridge@2.21.0/dist/median.min.js"></script>
```

## Quick Reference

| Task | Exact call |
|---|---|
| Page-load command (library) | Define `function median_library_ready() { median.module.fn(...) }` |
| Cover already-initialized case | `if (window.median) { window.median_library_ready(); }` |
| Import (NPM) | `import Median from "median-js-bridge";` |
| Ready hook (NPM) | `Median.onReady(() => { ... });` |
| In-app check (NPM) | `Median.isNativeApp()` → `true` / `false` |
| Platform (NPM) | `Median.getPlatform()` → promise of `'web' \| 'android' \| 'ios'` |
| Frontend detection (any mode) | `navigator.userAgent.indexOf('median') > -1` |
| iOS / Android split | `navigator.userAgent.indexOf('MedianIOS')` / `('MedianAndroid')` |
| Simple protocol call | `<a href="median://statusbar/set?style=light">…</a>` |
| Complex protocol call | `window.location.href = 'median://…'` |
| Chain commands | `median://nativebridge/multi` with an array of URLs |
| Add listener (NPM) | `const id = Median.appResumed.addListener(() => {...})` |
| Remove listener (NPM) | `Median.appResumed.removeListener(id)` |
| SPA soft navigation | `Median.jsNavigation.url.addListener((url) => {...})` |
| Legacy app prefix | `gonative.statusbar.set()` / `gonative://statusbar/set` |
| Legacy full command map | [references/gonative-legacy.md](references/gonative-legacy.md) — every `gonative.*` family, iOS/Android-exclusive commands included |

## How It Works

### Option 1 — Injected library

The library is injected into the page DOM at runtime and is available on any page displayed in the app. It initializes asynchronously, so gate page-load commands behind `median_library_ready()` — and call it manually if the library may already have initialized before your script defines the function:

```html
<script>
  function median_library_ready() {
    // Your bridge calls here, e.g.:
    median.navigationTitles.setCurrent({ title: "Your Title" });
  }
  if (window.median) {
    window.median_library_ready();
  }
</script>
```

See [references/library-and-protocol.md](references/library-and-protocol.md) for the full pattern, command chaining, and the GoNative naming fallback.

### Option 2 — NPM package (`median-js-bridge`)

Framework-agnostic package for SPAs (React, Angular, Vue, …). The import is **capitalized `Median`**; lowercase `median` is reserved for the injected library. When using the package, disable library injection in App Studio → Website Overrides.

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

Core entry points: `Median.onReady(callback)`, `Median.isNativeApp()`, `Median.getPlatform()` (promise → `'web' | 'android' | 'ios'`). See [references/npm-package.md](references/npm-package.md) for listeners, `jsNavigation.url` SPA navigation, and best practices.

### Option 3 — `median://` protocol

Direct, lower-level calls without the library. Use an HTML anchor `href` for simple commands or `window.location.href` for complex ones. Parameters containing JSON or special characters must be `encodeURIComponent()`-encoded. To run several commands, chain them through `nativebridge/multi` with an array of URLs — sequential separate calls can end up with only the last command executing.

```html
<a href="median://statusbar/set?style=light">Set status bar</a>
```

```javascript
window.location.href = 'median://run/median_device_info?callback=deviceInfoCallback';
```

### Detecting app vs. browser

Every request from the app carries an appended user-agent token — `MedianIOS/1.0 median` (iOS) or `MedianAndroid/1.0 median` (Android), with legacy `GoNativeIOS/1.0 gonative` / `GoNativeAndroid/1.0 gonative` equivalents:

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    // Running inside the Median app
    median.module.command({ 'parameter': 'value' });
    document.querySelector('.webNav').style.display = 'none';
    document.querySelector('.appOnly').style.display = 'block';
}
```

Full strategy comparison (JS UA, server UA, custom HTTP headers, dedicated app URL) lives in [references/app-detection.md](references/app-detection.md).

### Iframe callbacks

Protocol commands accept `?callback=<globalFunctionName>`; the callback resolves against the **parent** window. The standard pattern: iframe triggers the protocol command, the parent page defines the global callback and relays the data back into the iframe via `postMessage`.

```html
<script>
    function deviceInfoCallback(deviceInfo) {
        const iframe = document.getElementById('iframe');
        const message = { action: 'populateTextarea', deviceInfo: deviceInfo };
        iframe.contentWindow.postMessage(message, '*');
    }
</script>
```

The complete parent + iframe markup is in [references/library-and-protocol.md](references/library-and-protocol.md) (note its security disclaimer before shipping it).

### Google Tag Manager

Median's GTM template injects `median.min.js` from unpkg in a sandboxed GTM tag, with optional `bridgeVersion` and `userAgentValue` fields. It pushes a `median_injected` dataLayer event (`"yes"`/`"no"`) on completion, which you can use to trigger subsequent tags:

```json
{ "event": "median_injected", "median_injected": "yes", "device_info": "[Value from userAgentValue field]" }
```

Config fields, sandbox permissions, and the failure payload are in [references/library-and-protocol.md](references/library-and-protocol.md).

## Pitfalls

- **`median_library_ready()` race** — the library initializes asynchronously. If it initialized *before* your page defines the function, the function is never invoked; always include the `if (window.median) { window.median_library_ready(); }` manual-call guard.
- **`'median is not defined'` on desktop is normal** — the injected library only exists inside the app. To develop/test outside the app, use the NPM package and disable library injection in App Studio → Website Overrides.
- **`Median` vs `median`** — capitalized `Median` is the NPM import; lowercase `median` is the injected library. Mixing them up fails silently or throws.
- **GoNative legacy apps** — apps last updated on GoNative.io must use `gonative` (e.g. `gonative.statusbar.set()` or `gonative://statusbar/set`). Apps updated on Median.co accept either prefix. The full legacy command map is in [references/gonative-legacy.md](references/gonative-legacy.md); its `navigationTitles.revert` and `navigationLevels.setCurrent` mappings carry an upstream-quirk flag — read the note there before porting those two calls.
- **SPA frameworks can't see `median`** — in React/Vue/Angular, `median` may not be in scope. Options: (1) expose callbacks globally (`window.callback_function = () => {}`); (2) use the NPM package with library injection toggled off; (3) use the `median://` protocol.
- **NPM without the App Studio toggle** — enabling *JavaScript Frameworks and NPM* is required; omitting the package while the toggle is on can cause unresponsive bridge functions and callbacks.
- **`@latest` in production** — pin the script-tag version (e.g. `@2.21.0`) and upgrade deliberately; `@latest` picks up every breaking change instantly.
- **Sequential protocol calls** — can race so only the last command runs; use `median://nativebridge/multi` with an array of URLs instead.
- **Protocol params need encoding** — any JSON/special characters in a `median://` URL must go through `encodeURIComponent()`.
- **SPA warm-start deep links** — without a `jsNavigation.url` listener, a warm-start deep link triggers a costly full page load, or no load at all if the app thinks the URL is already displayed.
- **Iframe security** — the docs' iframe-callback sample uses `postMessage(message, '*')` and a globally accessible callback; review with a security team and tighten the target origin for production.
- **Custom UA strings** — the UA token is configurable in App Studio → Website Overrides; verify detection live via the Device-Info bridge function before relying on it.

## Verification

1. Open https://median.dev/library-ready/ inside your app — the page confirms the library injected and `median_library_ready()` fired.
2. On desktop with the NPM route (library injection off in App Studio): the page loads with no `'median is not defined'` console error, and `Median.isNativeApp()` returns `false`.
3. Inside the app, run `navigator.userAgent.indexOf('median')` in the console — it returns a positive index; `indexOf('MedianIOS')` / `indexOf('MedianAndroid')` splits the platform.
4. DevTools → Network: the bridge script resolves to the pinned URL `https://unpkg.com/median-js-bridge@2.21.0/dist/median.min.js` — never `@latest`.
5. Tap an anchor `<a href="median://statusbar/set?style=light">` in the app — the status bar restyles without a page reload.
6. GTM route: after the tag fires, `dataLayer` shows `{ "event": "median_injected", "median_injected": "yes" }`.
7. Legacy GoNative app: `gonative.statusbar.set()` still restyles the bar; after the app updates through Median.co, `median.statusbar.set()` works too. Before porting `navigationTitles.revert` or `navigationLevels.setCurrent`, read the quirk note in [references/gonative-legacy.md](references/gonative-legacy.md) and verify the behavior against the live legacy build.

## References

- [references/library-and-protocol.md](references/library-and-protocol.md) — injected library, `median://` protocol, `nativebridge/multi`, ready timing, gonative naming, iframe callbacks, GTM template
- [references/npm-package.md](references/npm-package.md) — install, App Studio toggle, `onReady`/`isNativeApp`/`getPlatform`, listeners, `jsNavigation.url` SPA navigation
- [references/app-detection.md](references/app-detection.md) — UA strings, frontend/backend detection, custom headers, dedicated app URL, strategy table
- [references/gonative-legacy.md](references/gonative-legacy.md) — full verbatim GoNative-era `gonative.*` command map, legacy/migration only (upstream source page `https://docs.median.co/page/gonative-javascript-bridge` is offline; two mappings flagged as unverified quirks)
- [Repo recipes](../../recipes.md) — cross-skill recipes; this skill's app-vs-browser detection appears in the analytics-split and social-login-swap recipes
- Official docs: [JavaScript Bridge](https://docs.median.co/docs/javascript-bridge) · [Detecting App Usage](https://docs.median.co/docs/detecting-app-usage) · [NPM Package](https://docs.median.co/docs/npm-package) · [Basic Usage](https://docs.median.co/docs/basic-usage) · [Listeners](https://docs.median.co/docs/npm-package-usage-with-listeners) · [SPA Navigation](https://docs.median.co/docs/npm-package-spa-navigation) · [Iframe Callbacks](https://docs.median.co/docs/iframe-callbacks) · [Google Tag Manager](https://docs.median.co/docs/google-tag-manager) · Docs index: https://docs.median.co/llms.txt (append `.md` to any docs URL for markdown)
- Live demo pages: https://median.dev/library-ready/
