# Median Bridge — Injected Library & `median://` Protocol

Verbatim API detail from docs.median.co. Nothing invented; where the docs show a snippet, it is reproduced exactly.

Sources: https://docs.median.co/docs/javascript-bridge · https://docs.median.co/docs/iframe-callbacks · https://docs.median.co/docs/google-tag-manager

---

## Injected JavaScript Bridge Library

The library is injected into the page DOM at runtime and is available on any page displayed in the app. You call `median.module.function()` once the library is ready.

Some bridge commands return JS promises — handle with `async/await` or `.then()/.catch()`:

```javascript
median.iap.purchase({ productID: 'product_id' })
  .then(function(data) { /* handle success */ })
  .catch(function(error) { /* handle error */ });
```

### `median_library_ready()` timing

The library initializes asynchronously. For page-load commands, define `median_library_ready()` on the page; the bridge invokes it after initialization. If the library initialized *before* your page defines the function, call it manually:

```javascript
function median_library_ready() {
  // Your code here
}
if (window.median) {
  window.median_library_ready();
}
```

### GoNative → Median naming

- Apps last updated on **GoNative.io** must use `gonative`, not `median`: `gonative.statusbar.set()` or `gonative://statusbar/set`.
- Apps updated on **Median.co** may use either `median` or `gonative`.

### Other library gotchas

- `'median is not defined'` in the console is normal in a desktop browser — the library only exists inside the app. Use the NPM package to develop/test outside the app (and disable library injection in App Studio → Website Overrides).
- React/Vue/Angular: `median` may not be in scope. Options: (1) expose callbacks globally `window.callback_function = () => {}`; (2) use the NPM Package and toggle off library injection in App Studio → Website Overrides; (3) use the `median://` protocol.
- Developer demo page (open in-app to test): https://median.dev/library-ready/

---

## `median://` Protocol

Direct, lower-level calls without the library.

- Simple commands: HTML anchor `href`
- Complex commands: `window.location.href`
- Parameters with JSON/special characters must be `encodeURIComponent()`-encoded.

```html
<a href="median://statusbar/set?style=light">Set status bar</a>
```

```javascript
window.location.href = 'median://run/median_device_info?callback=deviceInfoCallback';
```

### Command chaining — `nativebridge/multi`

Use `median://nativebridge/multi` with an **array of URLs** instead of sequential calls (sequential calls can cause only the last command to run).

---

## Bridge Callbacks Inside an Iframe

Purpose: access median callback data within an iframe. Pattern: the iframe triggers the protocol command with a named callback; the **parent** page defines that global callback function and relays data into the iframe via `postMessage`.

Protocol form: `median://run/<command>?callback=<globalFunctionName>` — the callback resolves against the **parent** window.

### Parent setup (verbatim from docs)

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

### Iframe setup (verbatim from docs)

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
            document.getElementById('deviceInfoOutput').value = JSON.stringify(event.deviceInfo);
        }
    });
</script>
```

> Security disclaimer from the docs: the parent page and the callback function may be publicly accessible; review with a security team and follow best practices to prevent data leaks. The sample uses `postMessage(message, '*')` — tighten the target origin for production.

---

## Google Tag Manager Template

Median's GTM template connects a site to the JS Bridge by injecting `median.min.js` from unpkg in a sandboxed GTM environment, with error handling, debug logs, and `dataLayer` status pushes.

### Config fields (both optional)

| Field | Purpose |
|---|---|
| `bridgeVersion` | Version number of the bridge to inject; blank defaults to latest |
| `userAgentValue` | Name of a GTM user-defined variable holding the device UA string; included in console logs and the `dataLayer` payload (Browser/OS/Device type) |

Constructed CDN URL: `https://unpkg.com/median-js-bridge@[version]/dist/median.min.js`

Required sandbox permissions:

- Inject scripts from `https://unpkg.com/`
- Access global variables (push to `dataLayer`)
- Log to console

### dataLayer events

Success payload:

```json
{ "event": "median_injected", "median_injected": "yes", "device_info": "[Value from userAgentValue field]" }
```

Failure payload:

```json
{ "event": "median_injected", "median_injected": "no", "device_info": "[Value from userAgentValue field]" }
```

Use the `median_injected` event to trigger subsequent tags.
