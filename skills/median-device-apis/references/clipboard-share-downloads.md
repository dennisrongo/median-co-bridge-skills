# Clipboard, Share Sheet, Downloads (full reference)

## median.clipboard.set / median.clipboard.get

Source: https://docs.median.co/docs/clipboard

**Purpose**: Write text to or read the device clipboard from the web app.

### Call signatures

```javascript
median.clipboard.set({ data: "..." })
median.clipboard.get()                              // promise
median.clipboard.get({ callback: "fnName" })        // named callback
```

### Parameters

| Call | Param | Type | Required | Description |
|---|---|---|---|---|
| set | `data` | string | Required | The text string to write to the clipboard |
| get | `callback` | string (function name) | Optional | Named callback for legacy/native contexts |

### Response shape (both get patterns)

| Property | Type | Description |
|---|---|---|
| `data` | string | Clipboard text content. Present on success. |
| `error` | string | Error message. Present when clipboard could not be read. |

### Platform notes

- iOS 14+ may show an OS banner that the app accessed the clipboard (cannot be suppressed).
- Android 10+ may restrict clipboard reads when the app is not in foreground — trigger reads from explicit user action.

### Minimal snippet

```javascript
median.clipboard.set({ data: "PROMO2024" });
median.clipboard.get().then(function (result) {
  if (result.data) { console.log("Clipboard contents:", result.data); }
  else { console.error("Clipboard error:", result.error); }
});
```

---

## median.share.sharePage (Prompt Share Dialogue)

Source: https://docs.median.co/docs/prompt-share-dialogue

**Purpose**: Trigger the native iOS/Android share dialog to share the current URL (or a given URL) plus optional text.

### Call signatures

```javascript
median.share.sharePage();
median.share.sharePage({url: 'https://median.co/about'});
median.share.sharePage({url: 'https://median.co/about', text: 'Visit Median here'});
```

### Parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `url` | string | Optional | Defaults to current URL |
| `text` | string | Optional | Shared along with URL |

### Callback/promise

None documented.

### Platform notes

- Callable from JS context including native tab menus/sidebars.
- Share options depend on installed apps; simulators may show limited options.

---

## median.share.downloadFile / median.share.downloadImage (Download File)

Source: https://docs.median.co/docs/download-file

**Purpose**: Download (and optionally open) a file on the user's device, or save an image to the photo gallery.

### Call signatures

```javascript
median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true|false})
median.share.downloadImage({url: 'https://yoursite.com/file.jpg'})
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `url` | string | Public URL required — localhost NOT supported |
| `open` | boolean | **Android-only parameter** — shows the "Open with" dialog after download |

### Callback/promise

None documented.

### Platform behavior matrix

| Context | Behavior |
|---|---|
| iOS | File downloads, then is passed to the native "Open with" system dialog. |
| Android + "Downloads Folder Enabled" (Permissions tab) | Downloads silently; `open: true` shows the "Open with" dialog. |
| Android + "Private to App" enabled (Permissions tab) | Always shows the "Open with" dialog after download. |

### Gotchas

- URLs must be publicly available.
- Alternative hands-off path: set `Content-Disposition: inline` (view in webview) vs `attachment` (download) headers on your server.
- PDFs view in-app via Apple PDFKit (iOS) / Pdf-Viewer library (Android); both viewers include a Print icon (iOS bottom-right, Android top-right).

### Minimal snippet

```javascript
median.share.downloadFile({url: 'https://yoursite.com/file.pdf', open: true});
median.share.downloadImage({url: 'https://yoursite.com/file.jpg'});
```
