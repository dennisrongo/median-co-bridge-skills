# Keyboard State, App Resumed, Clear Webview Cache (full reference)

## median.keyboard (Keyboard State Tracking)

Source: https://docs.median.co/docs/keyboard-state-tracking

**Purpose**: Get/subscribe to on-screen keyboard visibility and size; toggle iOS keyboard accessory view.

### Call signatures

```javascript
median.keyboard.info({'callback': function})   // current state; optional callback, otherwise returns a Promise
median.keyboard.listen(function)               // subscribe to state changes
median.keyboard.listen("")                     // stop listening
median.keyboard.showAccessoryView(true|false)  // iOS only
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `callback` | function | Optional for `info` |
| listener | function | For `listen`; pass `""` to stop |
| `true`/`false` literal | boolean | For `showAccessoryView` — `true` shows, `false` hides the iOS accessory view; the docs pass bare literals and document no parameter name |

### Response shape (both `info` and each `listen` event — verbatim)

```javascript
{
  'visible': BOOL,
  'keyboardWindowSize': { 'width': INT, 'height': INT },
  'visibleWindowSize': { 'width': INT, 'height': INT }
}
```

### Platform notes

- Accessory view toggle is iOS-only (spell check/formatting toolbar above keyboard).
- `visibleWindowSize` = remaining visible area with keyboard shown — use it to reposition sticky footers/CTAs.

### Minimal snippet

```javascript
function hideKeyboard(data) {
  if (data.visible) { median.tabNavigation.setTabs({ enabled: false }); }
  else { median.tabNavigation.setTabs({ enabled: true }); }
}
median.keyboard.listen(hideKeyboard);
median.keyboard.showAccessoryView(false); // iOS only
```

---

## median_app_resumed() (App Resumed Callback)

Source: https://docs.median.co/docs/app-resumed-callback

**Purpose**: Page-defined function automatically invoked when the app resumes from background — use for data refresh, content updates, or full reload.

### Call signature

You define the function on the page (or via Custom JavaScript); native invokes it. There is **no `median.*` call**.

```javascript
function median_app_resumed() { window.location.reload(); }
```

- **Parameters**: none.
- **Callback shape**: none — it IS the callback.

### Platform notes

- On Android, `median_app_resumed()` is also triggered after any native permission prompt.
- For kiosk/signage auto-reload, Median offers a separate native plugin (sales contact required) that bypasses JavaScript.

### Gotchas

- Must be present on the current page.
- NPM package alternative: `Median.appResumed.addListener(() => {...})`.

---

## median.webview.clearCache (Clear Webview Cache)

Source: https://docs.median.co/docs/clear-webview-cache

**Purpose**: Programmatically clear the webview cache to force fresh asset loads (dev/troubleshooting).

### Call signature

```javascript
median.webview.clearCache();
```

- **Parameters**: none. **Callback/promise**: none documented.

### Android-only appConfig.json options (under `general`)

```json
"general": { "androidClearCache": true }
"general": { "androidCacheMode": "no_cache" }
```

| Mode | Description |
|---|---|
| `no_cache` | Always load from the network, never use the cache. |
| `cache_only` | Only use cached content, never hit the network. |
| `cache_else_network` | Use cache if available, otherwise fetch from the network. |
| `default` | Use the system's default caching behavior. |

### Gotchas

- Don't call on every app start — kills perf by re-downloading assets.
- Production fix is asset versioning (e.g. `application_782374982.js`) + correct cache headers.
- Docs don't state whether cookies/localStorage/sessions are affected — test login behavior before production use.
