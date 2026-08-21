# Device Info — `median.deviceInfo()` (full reference)

Source: https://docs.median.co/docs/device-info

## Purpose

Returns real-time device, app, and advertising-identifier data for analytics, compatibility checks, support, and attribution.

## Call signatures (all three work)

```javascript
// 1. Auto-invoked callback — native calls this after EVERY page load
function median_device_info(deviceInfo) { console.log(deviceInfo); }

// 2. Manual trigger of your median_device_info() function
median.run.deviceInfo();

// 3. Promise form
median.deviceInfo().then(function (deviceInfo) { /* ... */ });
// or
var deviceInfo = await median.deviceInfo();
```

- **Parameters**: none on the input side.
- The auto-callback **must be defined at page-load time**; it cannot be added asynchronously or deferred.

## Response shape (verbatim from docs)

```javascript
{
  platform: 'ios',                                   // 'ios' | 'android'
  "SHA-1": 'XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX',    // only on Android (signing cert fingerprint)
  appId: 'io.median.example',
  appVersion: '1.0.0',
  appBuild: '1.0.0',                                 // appVersionCode (number) on Android
  carrierNames: ['AT&T'],                            // Android requires READ_PHONE_STATE
  distribution: 'release',
  hardware: 'armv8',
  installationId: 'xxxx-xxxx-xxxx-xxxx',
  apnsToken: '',                                     // only on iOS
  language: 'en',
  model: 'iPhone',
  os: 'iOS',
  osVersion: '10.3',
  timeZone: 'America/New_York',
  isFirstLaunch: false,                              // true on first launch of the app
  idfa: '00000000-0000-0000-0000-000000000000',      // iOS only; only if ad/attribution SDK configured
  gaid: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'      // Android only; only if ad/attribution SDK configured
}
```

## Field reference

| Key | Type | Platform | Description |
|---|---|---|---|
| `platform` | string | Both | `'ios'` / `'android'` |
| `os` | string | Both | OS name |
| `osVersion` | string | Both | OS version |
| `appId` | string | Both | App bundle ID |
| `appVersion` | string | Both | App version string |
| `appBuild` | string/number | Both | Numeric `appVersionCode` on Android |
| `installationId` | string | Both | Per-installation identifier |
| `distribution` | string | Both | e.g. `release` |
| `hardware` | string | Both | e.g. `armv8` |
| `model` | string | Both | Device model |
| `carrierNames` | array | Both | Android requires `READ_PHONE_STATE` |
| `timeZone` | string | Both | IANA timezone |
| `language` | string | Both | Device language |
| `isFirstLaunch` | boolean | Both | `true` on first launch of the app |
| `SHA-1` | string | Android-only | Signing certificate fingerprint |
| `apnsToken` | string | iOS-only | APNs push token |
| `idfa` | string | iOS-only | Requires ad SDK configured |
| `gaid` | string | Android-only | Requires ad SDK configured |

## Platform notes

- `gaid`/`idfa` are only returned when the **Adjust, AppsFlyer, or Meta App Events (iOS-only)** plugin is configured; otherwise the fields are omitted.
- **IDFA requires ATT permission** — until granted, `idfa` returns the zeroed placeholder `00000000-0000-0000-0000-000000000000`. Always compare against that placeholder before using for attribution.
- **GAID** is omitted on devices without Google Play Services (e.g. Huawei), is user-resettable, and is increasingly OS-restricted.

## Gotchas

- `median_device_info()` must exist at page load (auto-invoked after every page load).
- NPM library helper: `Median.getPlatform()` returns a promise resolving to `'web'`, `'ios'`, or `'android'`.

## Minimal snippet

```javascript
function median_device_info(deviceInfo) { console.log(deviceInfo); }
median.run.deviceInfo();
var deviceInfo = await median.deviceInfo();
```
