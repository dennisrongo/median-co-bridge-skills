# Geolocation + Camera & File Upload (full reference)

## Geolocation — `median_geolocation_ready()` / `median.android.geoLocation.*`

Source: https://docs.median.co/docs/location-services

**Purpose**: Use `navigator.geolocation` inside the app without iOS's double permission prompt, and request Android location permission at runtime.

### iOS: avoid duplicate geolocation prompts

On iOS, location permission is requested automatically at runtime — but the app must delay geolocation API calls until native location services are fully initialized, or the user is prompted **twice** (once from the native app, once from the webview). Median's native iOS layer calls the `median_geolocation_ready()` JavaScript function once location services are initialized; routing the first call through it ensures the web-based geolocation API uses the native implementation.

### Call signatures (verbatim from docs)

```javascript
// Automatically called by Median when iOS native location services are ready
function median_geolocation_ready() {
  navigator.geolocation.getCurrentPosition(locationSuccess, locationError, locationOptions);
}

// Fallback: Call immediately on non-iOS platforms
if (!navigator.userAgent.includes('MedianIOS')) {
  median_geolocation_ready();
}
```

Android — request runtime permission through the bridge (requires Location Services permission enabled in app configuration):

```javascript
median.android.geoLocation.promptLocationServices();

median.android.geoLocation.isLocationServicesEnabled({'callback':function});
// Return value:
{
  "enabled": true | false
}
```

### Parameters

| Call | Param | Type | Description |
|---|---|---|---|
| `promptLocationServices` | — | — | No parameters; shows the native permission dialog. No effect if permission is already granted. |
| `isLocationServicesEnabled` | `callback` | function | Receives the return value `{ "enabled": true \| false }` |

### Platform notes

- iOS: the shim means standard `navigator.geolocation` calls route through the native implementation — there is no separate Median geolocation API to learn.
- Android: the documented form of `isLocationServicesEnabled` passes a `{'callback': function}` — the docs do not describe a promise form. `promptLocationServices()` requires the **Location Services permission enabled in your app configuration** first.
- Continuous **Background Location** is a separate native plugin ([docs](https://docs.median.co/docs/background-location)) — not part of this core-bridge API.

### Gotchas

- `median_geolocation_ready()` follows the same page-load rule as `median_device_info()`: define it synchronously — a definition added late (SPA async mount) can miss the native invocation.
- The user-agent check is `navigator.userAgent.includes('MedianIOS')` — copy it exactly (capital `M`, `IOS`).

### Live demo

https://median.dev/location-services/

---

## Camera & file upload — `median.camera.setCaptureQuality` / `saveToGallery`

Sources: https://docs.median.co/docs/camera-and-file-uploads · https://docs.median.co/docs/web-rtc

**Purpose**: Standard `<input type="file">` upload flows, plus Android-only bridge control of camera capture quality and gallery storage.

### Call signatures (verbatim from docs)

```javascript
// Set camera capture quality to low
median.camera.setCaptureQuality("low")

// Set camera capture quality to high (default)
median.camera.setCaptureQuality("high")

// Prevent captured media from being saved to the gallery
median.camera.saveToGallery(false)

// Re-enable gallery saving (default behavior)
median.camera.saveToGallery(true)
```

### Parameters

| Call | Param | Type | Description |
|---|---|---|---|
| `setCaptureQuality` | quality literal | `"high" \| "low"` | `"high"` — maximum resolution and file size (default); `"low"` — reduced resolution to minimize file size |
| `saveToGallery` | boolean literal | `true \| false` | Default `true` — captured photos/videos are saved to the device media gallery; `false` disables. **This preference persists across sessions once set.** |

Both calls are **Android-only** — the docs flag the camera capture configuration options as "Android-specific and are not supported on iOS".

### Platform notes

- File and image upload via standard `<input type="file">` is natively supported in both the iOS and Android WebViews — no additional plugins or configuration required; the system presents the file picker / camera options automatically.
- iOS virtual device simulators have camera functionality disabled by security policy — test camera capture on the Android simulator or a real device.
- WebRTC (`navigator.mediaDevices.getUserMedia()`) camera/microphone permission prompts are handled natively on both platforms — no native code or extra configuration; WebRTC is restricted on insecure origins, so serve over **HTTPS**.

### Gotchas

- `saveToGallery(false)` persists across sessions — deliberately re-enable with `true` when gallery saving should return.
- Choose `"low"` when upload speed or server storage costs matter; `"high"` when image fidelity matters.

### Live demos

- https://median.dev/camera/ (file + camera uploads)
- https://median.dev/camera-settings (Android camera settings)

### Minimal snippet

```html
<input type="file" accept="image/*">
<script>
  median.camera.setCaptureQuality("low");
  median.camera.saveToGallery(false);
</script>
```
