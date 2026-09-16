# Scanning (full reference)

Three plugin-gated capture families: QR/barcode (`median.barcode.*`), document scanning (`median.documentScanner.scanPage`), and NFC tag reading (`median.nfc.*`). All require the JavaScript Bridge enabled.

## median.barcode.* (QR / Barcode Scanner) — plugin required

Source: https://docs.median.co/docs/qr-barcode-scanner

**Purpose**: Scan QR codes and barcodes with the device camera — in-store retail, warehouse/inventory management, data-center asset tracking. Built on customized open-source libraries (Android: ZXing `github.com/zxing/zxing`; iOS: hyperoslo `github.com/hyperoslo/BarcodeScanner`) with full functionality, no licensing fees, no usage limits.

**Prerequisite**: QR / Barcode Scanner plugin enabled under Native Plugins (App Studio).

### Call signatures

```javascript
// Callback style
median.barcode.scan({ formats: ['QR_CODE', 'EAN_13'], callback: process_barcode });

// Promise style
median.barcode.scan({ formats: ['QR_CODE'] }).then(function (data) {
  if (data.success) { alert('Got barcode: ' + data.code); }
});

// All supported formats (default — omit `formats` or pass it empty)
median.barcode.scan().then(function (data) { ... });

// Runtime prompt override
median.barcode.setPrompt('Align the QR code or barcode within the frame to scan automatically');
```

The native scanner UI launches in-app; on completion focus returns to the webview and the result object is delivered.

### Parameters

| Param | Type | Description |
|---|---|---|
| `formats` | string[] | Restrict which code types can trigger a scan; omitted/empty = all supported formats |
| `callback` | function | Receives the scan result (callback style) |

### Response shape

```javascript
{
  success: true,      // boolean — true if a code was scanned
  type: "STRING",     // Symbology (e.g. 'qr', 'code128')
  code: "STRING",     // The actual scanned value
  error: "STRING"     // Error message if unsuccessful
}
```

### Supported formats

| Kind | Formats |
|---|---|
| 2D (QR codes) | `QR_CODE`, `DATA_MATRIX`, `AZTEC`, `PDF_417` |
| 1D (barcodes) | `EAN_13`, `EAN_8`, `UPC_A`, `UPC_E`, `CODE_39`, `CODE_93`, `CODE_128`, `ITF`, `CODABAR` |

### Prompt

- Static: set in App Studio → Native Plugins → QR / Barcode Scanner (custom prompt message).
- Runtime: `median.barcode.setPrompt('...')` overrides dynamically in-app.

### Gotchas

- Omitting `formats` means every supported symbology can trigger — with multiple code types in view, the wrong code can win; restrict formats when targeting one symbology (format targeting added to the scanner Jul 2026).
- The result arrives only after the scanner UI closes on a successful read — there is no continuous/queue mode.

### Minimal snippet

```javascript
median.barcode.scan({ formats: ['QR_CODE'] }).then(function (data) {
  if (data.success) { console.log(data.type, data.code); }
});
median.barcode.setPrompt('Align the code within the frame');
```

---

## median.documentScanner.scanPage (Document Scanner) — plugin required

Source: https://docs.median.co/docs/document-scanner

**Purpose**: Scan documents with the device camera — automatic edge detection, angle/perspective/aspect-ratio correction, text clarity and sharpness enhancement, clean cropped export as base64 JPEG. Attach to a form or upload to your server asynchronously.

**Prerequisite**: Document Scanner plugin enabled under Native Plugins (App Studio).

### Call signature

```javascript
median.documentScanner.scanPage({'callback':CALLBACKFUNCTION});
// docs' literal form — replace CALLBACKFUNCTION with your function name
```

### Callback payload

```javascript
{
  image: "base64-encoded string",  // scanned JPEG image
  mimeType: "image/jpeg",
  encoding: "base64"
}
```

### Gotchas

- Output is a base64 JPEG only — no other format or encoding is returned.
- Full-page scans are large; upload asynchronously rather than inlining into URLs.

### Minimal snippet (docs example)

```html
<script>
function cb(data) {
  document.getElementById('page').setAttribute('src', 'data:' + data.mimeType +
    ';' + data.encoding + ', ' + data.image);
}
</script>
<p><a onclick="median.documentScanner.scanPage({'callback':cb});">Scan page</a></p>
<p><img id="page" src=""/></p>
```

---

## median.nfc.* (NFC Tag Scanner) — plugin required

Source: https://docs.median.co/docs/nfc-tag-scanner

**Purpose**: Read NFC tags on iOS and Android while the app is open; on supported devices, background tag reading works for apps with universal links.

**Prerequisite**: NFC Tag Scanner plugin enabled in App Studio before rebuilding. If you build from downloaded iOS source or manually manage Apple signing assets, confirm the **Near Field Communication Tag Reading capability** is enabled in Xcode and in your Apple Developer account.

### Call signatures

```javascript
// Availability — promise or callback
median.nfc.status().then(function (data) { if (data.available) { ... } });
median.nfc.status({'callback': nfcStatusCallback});

// Tag reading — promise or callback
median.nfc.readTag({
  'message': STRING,    // Android only — shown in the prompt while scanning
  'openUrl': BOOLEAN    // TRUE + an http/https URL read → opened automatically
}).then(function (data) { ... });
median.nfc.readTag({ 'callback': readTagCallback, 'message': STRING });
```

### Parameters (readTag)

| Param | Type | Description |
|---|---|---|
| `message` | String | Android only — prompt text shown while scanning for tags |
| `openUrl` | Boolean | If TRUE and the tag returns an http/https URL, the URL opens automatically |
| `callback` | function | Receives the result (callback style) |

### Response shape (readTag)

```javascript
{
  success: BOOLEAN,   // true if scanning was successful
  cancel: BOOLEAN,    // true if the user canceled the NFC scan prompt
  error: STRING,      // "SystemIsBusy", "SessionTimeout", "SessionTerminatedUnexpectedly", or "Error"
  type: STRING,       // NFC tag type
  prefix: STRING,     // NDEF tag prefix, e.g. 'https://'
  content: STRING,    // rest of the tag payload after the prefix
  uri: STRING         // concatenation of prefix + content
}
```

`status()` → `{ available: BOOLEAN }`.

### Platform differences

| Platform | Behavior |
|---|---|
| iOS | Default system NFC UI; source builds need the NFC Tag Reading capability (see prerequisite) |
| Android | Custom native UI providing user feedback, shows the `message` prompt text |

### Gotchas

- Check `error` → `cancel` → `success`/`uri` in that order; a user cancel is not an error and not a success.
- `message` is ignored on iOS.

### Minimal snippet

```javascript
median.nfc.readTag({ message: 'Hold your device near the storage bin' }).then(function (data) {
  if (data && data.error) { console.log('There was an error scanning for NFC tags'); return; }
  if (data && data.uri) { /* use data.uri */ }
});
```
