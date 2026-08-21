# Reader Modal + Secure Modal + App Review (full reference)

## median.readerModal (Reader Modal) — iOS, Apple entitlement required

Source: https://docs.median.co/docs/reader-modal

**Purpose**: Open an external-link account-management/payments modal for Reader apps (magazines, books, audio, video, etc.) per Apple's Reader app policy.

**Prerequisite**: Apple **External Link Account entitlement** permission (special approval from Apple).

### Call signatures

```javascript
const { canMakePayments } = await median.readerModal.canMakePayments();
median.readerModal.showModal();
```

### Parameters

None.

### Response shape

`canMakePayments()` resolves `{ canMakePayments: BOOL }` (per destructuring in docs).

### Platform notes

- Modal appears on iOS 16+ devices.
- Always verify payment ability before showing the modal.

---

## median.modal.launch (Secure Modal) — plugin required

Source: https://docs.median.co/docs/secure-modal

**Purpose**: Open a secure iOS WKWebView window with external scripting blocked (required by e.g. Apple Pay JS API).

### Call signature

```javascript
// callback method
median.modal.launch({ 'url': 'https://applepaydemo.apple.com', 'autoClosePath': '/payment-complete', 'callback': modal_closed })

// promise method — same object without callback, chained .then(function (data) {...})
```

### Parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `url` | string | Required in examples | URL to open in the secure modal |
| `autoClosePath` | string | Optional | Auto-closes modal when that URL path loads (e.g. after successful payment) |
| `callback` | function | Optional | Receives the close payload |

### Response shape (returned when modal closes — verbatim)

```javascript
{
  closeMethod: "closeButton" | "autoClosePath",
  url: STRING,            // URL with complete path shown when closed
  params: {               // Params on the URL when closed e.g. ?status=success results in status: success
    key: value,
    key2: value
  }
}
```

### Platform notes / Gotchas

- On iOS 16+, Apple permits Apple Pay directly in an app webview — optionally check iOS version via `deviceInfo` and skip the modal there.

### Minimal snippet

```javascript
median.modal.launch({
   'url': 'https://applepaydemo.apple.com',
   'autoClosePath':'/payment-complete',
   'callback': modal_closed
});
function modal_closed(data) {
  if (data.closeMethod == 'closeButton') { alert('modal closed via button at: ' + data.url); }
  else if (data.closeMethod == 'autoClosePath') { alert('modal closed automatically'); }
}
```

---

## median.appreview.prompt (App Review) — plugin required

Source: https://docs.median.co/docs/app-review

**Purpose**: Prompt the user to rate/review the app on the Apple App Store or Google Play.

**Prerequisite**: App Review plugin installed. iOS config: set your App Store ID (numerical `idXXXXXXX` from the listing URL; e.g. YouTube = `544007664`).

### Call signature

```javascript
median.appreview.prompt({ callback: appReviewComplete })
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `callback` | function | Optional — run once the review modal has been closed |

### Platform notes / Gotchas

- **iOS**: Apple controls whether the prompt actually displays (anti-spam); dev builds always show it; **no effect in TestFlight builds**.
- **Android**: Google Play enforces a time-based rate-limit quota on review prompts.
- Best practices: avoid direct CTA buttons, target natural completion points, fall back to redirecting to the Play Store listing when quota likely exceeded.
