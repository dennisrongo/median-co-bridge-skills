---
name: median-scanning
description: "Scan QR codes and barcodes, scan documents with auto-crop and image enhancement, and read NFC tags — barcode scanning, document scanner, NFC tag reader. Use when a Median app needs camera-based code/document scanning, NFC tag reads, or offline scan-and-sync capture."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/scanning
---

# Median Scanning

Camera and NFC capture in a Median-built iOS/Android app, driven from web JS through the JavaScript Bridge: QR/barcode scanning with format restriction and a customizable prompt, a document scanner that detects edges and returns an enhanced base64 JPEG, and NFC tag reading with URL auto-open. Every API in this skill requires the **JavaScript Bridge enabled** *plus* its specific **Native Plugin** (QR / Barcode Scanner, Document Scanner, or NFC Tag Scanner) toggled on in App Studio — calls silently do nothing when the plugin is off. Scanned values persist offline via `median.storage.app` (see `median-native-features` and [Repo recipes](../../recipes.md)).

## When to Use

- Scanning QR codes or barcodes — retail, inventory, warehouse, asset tracking
- Restricting scans to specific symbologies (`formats: ['QR_CODE', 'EAN_13']`) when several code types are in view
- Customizing the scanner prompt message at runtime (`setPrompt`) or in plugin settings
- Turning paper documents into clean, cropped, enhanced JPEG scans for form uploads
- Reading NFC tags (NDEF URIs) and optionally auto-opening tagged URLs
- Building offline capture: scan → store on device → sync when back online ([Repo recipes](../../recipes.md))

Don't use for:

- Camera photo capture, capture quality, save-to-gallery — `median-device-apis`
- Storing scanned values (App/Cloud Storage), background audio/media playback — `median-native-features`
- Push notifications, biometrics — `median-push-auth`
- Crashlytics, in-app purchases — `median-analytics-iap`

## Prerequisites

- **JavaScript Bridge enabled** in App Studio (required for every bridge call).
- **Native Plugins** — each feature needs its plugin enabled in App Studio → Native Plugins, and the app rebuilt:

| Feature | Plugin / gate |
|---|---|
| QR / Barcode | QR / Barcode Scanner plugin — no licensing fees or usage limits (ZXing on Android, hyperoslo BarcodeScanner on iOS) |
| Document Scanner | Document Scanner plugin |
| NFC | NFC Tag Scanner plugin; when building from downloaded iOS source or managing Apple signing yourself, the Near Field Communication Tag Reading capability must be enabled in Xcode and your Apple Developer account |

Scandit QR / Barcode (licensed enterprise variant) and iBeacon (proximity) exist as separate plugins outside this skill's scope.

## Quick Reference

| Task | Exact call |
|---|---|
| Scan a QR/barcode (all formats) | `median.barcode.scan().then(function (data) { if (data.success) alert(data.code); });` |
| Restrict scanned formats | `median.barcode.scan({ formats: ['QR_CODE', 'EAN_13'], callback: fn });` |
| Set scanner prompt at runtime | `median.barcode.setPrompt('Align the QR code or barcode within the frame to scan automatically');` |
| Scan a document | `median.documentScanner.scanPage({ callback: fn })` → `{ image, mimeType: 'image/jpeg', encoding: 'base64' }` |
| Check NFC availability | `median.nfc.status()` → `{ available: BOOL }` |
| Read an NFC tag | `median.nfc.readTag({ message: 'Hold near the tag', openUrl: true })` → `{ success, cancel, error, type, prefix, content, uri }` |

## How It Works

### QR / barcode — `median.barcode.*`

```javascript
median.barcode.scan({
  formats: ['QR_CODE', 'EAN_13'],   // omit or leave empty = all supported formats
  callback: process_barcode
});

function process_barcode(data) {
  if (data.success) {
    alert('Got barcode: ' + data.code);   // data.type holds the symbology, e.g. 'qr', 'code128'
  }
}

median.barcode.setPrompt('Align the QR code or barcode within the frame to scan automatically');
```

Response: `{ success: BOOL, type: STRING, code: STRING, error: STRING }`. Supported formats — 2D: `QR_CODE`, `DATA_MATRIX`, `AZTEC`, `PDF_417`; 1D: `EAN_13`, `EAN_8`, `UPC_A`, `UPC_E`, `CODE_39`, `CODE_93`, `CODE_128`, `ITF`, `CODABAR`. A static prompt can also be set in App Studio (Native Plugins → QR / Barcode Scanner); `setPrompt` overrides it at runtime.

### Document scanner — `median.documentScanner.scanPage`

```javascript
median.documentScanner.scanPage({ callback: function (data) {
  // data.image (base64), data.mimeType 'image/jpeg', data.encoding 'base64'
  document.getElementById('page').setAttribute('src',
    'data:' + data.mimeType + ';' + data.encoding + ', ' + data.image);
}});
```

The native UI detects document edges in real time, auto-corrects angle/perspective/aspect ratio, enhances text clarity, and exports a clean cropped scan as a **base64 JPEG** — attach it to a form or upload it to your server asynchronously.

### NFC — `median.nfc.*`

```javascript
median.nfc.status().then(function (data) {
  if (data.available) { /* show NFC-related UI */ }
});

median.nfc.readTag({
  message: 'Hold your device near the storage bin',   // Android only — prompt text
  openUrl: true                                        // auto-open an http(s) URL read from the tag
}).then(function (data) {
  if (data && data.error) { return console.log('NFC error: ' + data.error); }
  if (data && data.uri) { /* data.prefix + data.content */ }
});
```

Platform differences: iOS uses the default system NFC UI; Android shows a custom native UI carrying the `message` prompt. Background tag reading works on supported devices for apps with universal links. Errors: `SystemIsBusy`, `SessionTimeout`, `SessionTerminatedUnexpectedly`, `Error`. Full response shape and parameters in [references/scanning.md](references/scanning.md).

### Offline scan → store → sync

The QR scanner and App Storage both work without a network connection: launch the scanner while offline, persist the scanned value with `median.storage.app.set` (not Cloud Storage — cloud is unavailable offline), then read the key back and POST to your backend when connectivity returns. Full working HTML page in [Repo recipes](../../recipes.md).

## Pitfalls

- **Plugin-gated: enable before coding.** All three scanners silently fail without their Native Plugin enabled in App Studio and a rebuilt binary.
- **Omitting `formats` scans everything.** With several code types in view, any of them can trigger; pass `formats` to target specific symbologies (the format-targeting release, Jul 2026, exists for exactly this multi-code case).
- **Document scans arrive as base64 JPEG only** — full-page images are large; upload asynchronously, never inline them into URLs.
- **NFC `message` is Android-only** — iOS shows its default system UI and ignores the prompt text.
- **iOS source builds need the NFC capability** — the Near Field Communication Tag Reading capability must be present in Xcode and your Apple Developer account when building from downloaded source or managing signing manually.
- **`readTag` has three outcome shapes** — check `data.error` first, then `data.cancel` (user dismissed the prompt), then `data.success`/`data.uri`; don't treat a cancel as a successful empty scan.
- **Offline capture belongs to App Storage, not Cloud Storage** — cloud sync is unavailable offline; don't promise immediate durability.
- **Outside the app there is no `median` object.** Guard with `navigator.userAgent.indexOf("median") > -1` (see `median-bridge-setup`).

## Verification

1. Each scanner used: plugin enabled in App Studio, app rebuilt, and the matching demo page (median.dev/qr-barcode/ or median.dev/document-scanner/) behaves identically in-app.
2. QR scan of a printed QR code returns `{ success: true, code }` with the expected payload.
3. With `formats: ['EAN_13']`, a QR code in view does NOT trigger a scan; an EAN-13 barcode does.
4. `setPrompt('...')` — the custom message appears in the scanner UI on the next scan.
5. Document scan: the returned base64 renders in an `<img>` (`data:image/jpeg;base64, ...`) as a cropped, enhanced page.
6. `median.nfc.status()` reflects the device's NFC capability; a physical tag read returns `data.uri` = `prefix + content`; cancelling the prompt sets `data.cancel`.
7. Offline recipe: airplane mode → scan → `median.storage.app.set` → back online → `median.storage.app.get` returns the stored value for sync.

## References

- [references/scanning.md](references/scanning.md) — barcode formats and response shape, document scanner callback payload, NFC parameters/response/error values and platform differences
- [Repo recipes](../../recipes.md) — offline scan → store → sync capture pattern with the full HTML page
- Official docs: [Scanning](https://docs.median.co/docs/scanning) · [QR / Barcode Scanner](https://docs.median.co/docs/qr-barcode-scanner) · [Document Scanner](https://docs.median.co/docs/document-scanner) · [NFC Tag Scanner](https://docs.median.co/docs/nfc-tag-scanner)
- Live demo pages (open inside your app): https://median.dev/qr-barcode/ · https://median.dev/document-scanner/
