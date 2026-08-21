# Detecting App Usage (App vs. Browser)

Source: https://docs.median.co/docs/detecting-app-usage

---

## Default User-Agent Strings

Every HTTP request from the app carries an appended user-agent token:

| Platform | User agent string | Legacy user agent string |
| --- | --- | --- |
| iOS App | `MedianIOS/1.0 median` | `GoNativeIOS/1.0 gonative` |
| Android App | `MedianAndroid/1.0 median` | `GoNativeAndroid/1.0 gonative` |

> The UA string is configurable in App Studio → Website Overrides (see Custom User Agent). Confirm with the Device-Info bridge function in the live app.

---

## Frontend Detection (verbatim from docs)

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    // Running inside the Median app
    median.module.command({ 'parameter': 'value' });
    document.querySelector('.webNav').style.display = 'none';
    document.querySelector('.appOnly').style.display = 'block';
}
```

Platform-specific check for analytics:

```javascript
navigator.userAgent.indexOf('MedianIOS')    // iOS
navigator.userAgent.indexOf('MedianAndroid') // Android
```

---

## Backend Detection

Read the `User-Agent` header, e.g. in Node/Express:

```javascript
req.header('User-Agent').indexOf('median') > -1
```

---

## Detection Strategies Compared (from docs)

| Strategy | Best for | Complexity |
| --- | --- | --- |
| User-agent detection (JS) | Frontend UI changes, gating Bridge calls | Low |
| User-agent detection (server) | Serving different HTML, API responses | Low |
| Custom HTTP headers | Clean server-side detection, backend APIs | Low–Medium |
| Dedicated app URL | Fully separate app vs. browser experiences | Medium |

### Custom HTTP headers

Configured in App Studio under Web Overrides → Custom Headers, e.g. `X-App-Platform: median-ios`. Good for clean server-side detection and backend APIs.

### Dedicated app URL

Serve the app from a dedicated subdomain, e.g. `https://app.yoursite.com` — every request is definitionally from the app. Best when you want fully separate app vs. browser experiences.

---

## Best Practice

Wrap detection in a helper function (e.g. `logAnalyticsEvent(eventName, eventProperties)`) so app/browser branching stays centralized.
