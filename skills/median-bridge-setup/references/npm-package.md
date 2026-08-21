# Median NPM Package (`median-js-bridge`)

Framework-agnostic NPM package for using the Bridge in SPAs (React, Angular, Vue.js, etc.).

Sources: https://docs.median.co/docs/npm-package · https://docs.median.co/docs/basic-usage · https://docs.median.co/docs/npm-package-usage-with-listeners · https://docs.median.co/docs/npm-package-spa-navigation

---

## Install

```bash
npm install median-js-bridge --save
# or
yarn add median-js-bridge
```

Script tag alternative:

```html
<script type="text/javascript" src="https://unpkg.com/median-js-bridge@latest/dist/median.min.js"></script>
```

> Avoid `@latest` in production — pin a version (e.g. `@1.12.3`) and upgrade deliberately; `@latest` gets every breaking change instantly.

Links: https://github.com/gonativeio/median-javascript-bridge#readme · https://www.npmjs.com/package/median-js-bridge

### Required app config

Enable **Website Overrides > JavaScript Frameworks and NPM** in App Studio. This prevents conflicts with the injected Bridge Library and activates required listener support. Omitting the package while enabling the option may result in unresponsive Bridge functions and callbacks.

---

## Basic Usage

**`Median` not `median`** when using the NPM package — capitalized `Median` object; lowercase `median` is reserved for the runtime-injected library. Configure IDE type hinting for completion.

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

Promise-style example (OneSignal login + info, verbatim from docs):

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

---

## Listeners

Register listener functions that respond to native events. Events occurring *before* registration are queued and delivered once the listener initializes (reliable capture).

Call signature:

```javascript
const listenerId = Median.<interface>.addListener((data) => { ... });
// later:
Median.<interface>.removeListener(listenerId);
```

Each registration returns a `listenerId`.

### Documented listener interfaces (verbatim examples)

| Listener | Example |
| --- | --- |
| App resumed | `Median.appResumed.addListener(() => { console.log("App resumed listener triggered"); })` |
| OneSignal push opened | `Median.oneSignalPushOpened.addListener((data) => { console.log(JSON.stringify(data)); })` |
| Share into app | `Median.shareToApp.addListener((data) => { console.log(data.url, data.subject); })` |
| Device shake (Haptics) | `Median.deviceShake.addListener(() => { console.log("Device shake listener"); })` |
| AppsFlyer conversion data | `Median.appsFlyerConversionData.addListener((conversionDataMap) => { console.log(conversionDataMap.af_status); })` |

### Best practices (from docs)

- Store the `listenerId` if you'll remove it later.
- Register early (in `useEffect`/lifecycle hook).
- Remove listeners on unmount to avoid SPA memory leaks.

---

## SPA Navigation — `Median.jsNavigation.url`

Handle native-initiated navigation (tab bar taps, deep-link resume, push-notification resume) via the SPA router as soft loads instead of hard page loads.

```javascript
Median.jsNavigation.url.addListener((url) => {
  // Use your SPA's routing logic to navigate to the URL
  // Soft load the requested content, triggering a hard page load only if necessary
});
```

Triggered by:

- Native navigation button clicks (e.g. bottom tab bar)
- Deep-link app resume (email/SMS link launching app from background)
- Push notification app resume (tapped notification with embedded URL)

Gotcha: without the listener, a warm-start deep link causes a costly full page load — or worse, **no load at all** if the app incorrectly determines the URL is already displayed from the initial cold-start load.
