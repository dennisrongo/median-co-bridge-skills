# Screen Orientation (config-only) + Dark Mode (full reference)

## Screen Orientation — NO bridge command

Source: https://docs.median.co/docs/screen-orientation

**Purpose**: Lock or auto-rotate app orientation per OS/device type.

### Call signature

**None — App configuration only (App Studio setting).** There is no `median.*` bridge command for orientation.

### Modes

| Mode | Description |
|---|---|
| **Auto-Rotation** | Follow device rotation |
| **Fixed Portrait** | Lock portrait |
| **Fixed Landscape** | Lock landscape |

Modes are customizable by **Operating System** (iOS/Android) and **Device Type** (Phone/Tablet).

### Gotchas

- Enabling Fixed Portrait on iPads **automatically disables multi-tasking capabilities**.
- If asked to rotate/lock orientation from JavaScript: it cannot be done via the bridge — set the mode in app configuration and rebuild the app.

---

## median.screen.setColorScheme / resetColorScheme (Dark Mode)

Source: https://docs.median.co/docs/dark-mode

**Purpose**: Force Light/Dark/Auto color scheme for native UI (nav/tab/sidebar menus) and drive web content via `prefers-color-scheme`.

### Call signatures

```javascript
median.screen.setColorScheme("light")
median.screen.setColorScheme("dark")
median.screen.setColorScheme("auto")     // follow device mode
median.screen.resetColorScheme()         // revert to app-config default
```

### Parameters

| Param | Type | Description |
|---|---|---|
| (single argument) | string | `"light" \| "dark" \| "auto"` — a bare string, **not an options object**, per docs examples |

### Callback/promise

None documented.

### Platform notes

- **Web content dark mode**: the app sets `prefers-color-scheme` to `light`/`dark` dynamically with device mode.
- Median also sets a **`data-color-scheme-option`** property on the page with the current app setting (`light`, `dark`, or `auto`) — use it for building a toggle.
- Recommend CSS variables for light/dark palettes.

### Gotchas

- Docs wrap calls in `if (navigator.userAgent.indexOf("median") > -1)` to guard against running outside the app.

### Minimal snippet

```javascript
if (navigator.userAgent.indexOf("median") > -1) {
  median.screen.setColorScheme("dark");
  median.screen.resetColorScheme();
}
```
