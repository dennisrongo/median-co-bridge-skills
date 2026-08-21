# Share into App + Web Screenshot (full reference)

## median_share_to_app(data) (Share into App) — plugin required

Source: https://docs.median.co/docs/share-into-app

**Purpose**: Native share extension — users share URLs from any app into yours; app launches and invokes your JS callback with URL + subject.

**Prerequisite**: Share into App plugin enabled + **rebuild required** after enabling.

### iOS setup

- Configure an **App Group** identifier: enter the bare iOS Bundle ID, e.g. `com.example.app` — the platform derives `group.<bundleId>`.
- The App Group MUST be manually registered in the Apple Developer Portal and enabled on **both** the app App ID and the `<bundleId>.ShareExtension` App ID, or release builds fail.
- Also set the URL scheme protocol under **Link Handling** in App Studio.

### Call signature (page-defined callback invoked by native)

```javascript
function median_share_to_app(data) {
  alert(data.url);     // Shared URL
  alert(data.subject); // Page title or shared text
}
```

### Callback shape

| Field | Description |
|---|---|
| `data.url` | Shared URL |
| `data.subject` | Page title / shared text — not all share sources supply a subject |

### Gotchas

- Function must be defined on the website before the native callback fires (JS Bridge enabled).
- Plugin config changes require a new binary.
- NPM listener alternative: `Median.shareToApp.addListener((data) => { console.log(data.url, data.subject); })`.

---

## median.webScreenshot.* (Web Screenshot) — plugin required

Source: https://docs.median.co/docs/web-screenshot

**Purpose**: Capture and share visible web content or a specific element as an image, or export it as a Blob.

### Call signatures

```javascript
median.webScreenshot.shareScreen({ url: "...", text: "..." })       // capture + native-share visible webview area
median.webScreenshot.shareElement(element, { url: "...", text: "..." }) // capture + share a specific element (e.g. a <div>)
median.webScreenshot.captureScreen()                                 // returns screenshot as a Blob
median.webScreenshot.captureElement(element)                         // returns element screenshot as a Blob
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `element` | DOM element | For the Element variants |
| `url` | string | Optional context shared alongside the image |
| `text` | string | Optional context shared alongside the image |

### Callback/promise

- Share variants use the native share dialog.
- Capture variants return a Blob synchronously per docs examples.

### Gotchas

- `shareScreen` captures only the visible portion, excluding native UI (native navigation bars).
- Blob output can be uploaded to your server or fed to another plugin (docs cite Social Share).

### Minimal snippet

```javascript
median.webScreenshot.shareScreen({ url: "https://www.median.dev", text: "Median Developer Demo" });
const element = document.getElementById("content");
median.webScreenshot.shareElement(element, { url: "https://median.dev/", text: "Median Developer Demo" });
const screenShotblob = median.webScreenshot.captureScreen();
const elementBlob = median.webScreenshot.captureElement(element);
```
