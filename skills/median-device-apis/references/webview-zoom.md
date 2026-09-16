# Webview Zoom — `median.webview.getZoom` / `setZoom` (full reference)

Source: https://docs.median.co/docs/webview-zoom

**Purpose**: Configure the initial zoom level of the webview in App Studio and read/change it at runtime, to optimize content display for different screen sizes and user preferences.

### App configuration (required)

To use the webview zoom feature, set the **Viewport Width** to **"WebView Scale"** in the **Interface** settings of your app (App Studio > Interface > Viewport Width). Without this setting the zoom feature is not active.

The initial zoom level can also be set in App Studio during configuration; the bridge calls below adjust it dynamically at runtime.

### Call signatures (verbatim from docs)

```javascript
median.webview.getZoom();
median.webview.setZoom(1.1); // Sets zoom to 110%
```

### Parameters

| Call | Param | Type | Description |
|---|---|---|---|
| `setZoom` | zoom literal | number | Zoom multiplier — `1.1` sets 110% |
| `getZoom` | — | — | No parameters; retrieves the current zoom level |

No callback or promise return is documented for either call.

### Gotchas

- **Viewport Width gate**: Interface → Viewport Width must be "WebView Scale" or the zoom commands don't apply.
- **`median.webview.reload()` is not a documented API.** Community snippets show a `reload()` bridge command, but no current official docs page documents it (checked Webview Zoom, Clear Webview Cache, NPM Basic Usage, and Refresh Button — the Refresh Button page's own Pull-to-Refresh link 404s). Reload with `window.location.reload()`, the native top-nav Refresh Button, or Pull-to-Refresh (the latter two are App Studio configuration).

### Live demo

https://median.dev/webview-zoom/

### Minimal snippet

```javascript
median.webview.setZoom(1.1); // 110%
median.webview.getZoom();
```
