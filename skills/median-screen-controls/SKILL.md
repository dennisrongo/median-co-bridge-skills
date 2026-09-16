---
name: median-screen-controls
description: "Control screen brightness, keep-awake, Android fullscreen, dark/light color scheme, and Android swipe-navigation gestures; orientation is App Studio config only. Use when a task dims or wakes the screen, toggles fullscreen or swipe gestures, or forces dark/light mode inside a Median app."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/device-screen-control
---

# Median Screen Controls

Drive the physical screen experience from your web app at runtime: set brightness, keep the screen awake (wake lock), toggle Android fullscreen, force light/dark color scheme, toggle Android swipe-navigation gestures, and understand what orientation control is possible (spoiler: it is app-config only — there is no bridge command). These are core JavaScript Bridge APIs; no native plugins beyond the bridge itself — pinch-to-zoom and pull-to-refresh gesture defaults are App Studio configuration, not bridge calls.

Everything runs inside a Median-built iOS/Android app wrapping your existing website. In a desktop browser the `median` object does not exist; guard calls accordingly.

## When to Use

Reach for this skill when the task involves:

- Dimming or brightening the screen programmatically (e.g. for a reading or kiosk view), including restoring the previous brightness on navigation
- Preventing the screen from sleeping during video playback, recipes, workouts, guided flows
- Hiding Android status/navigation bars for an immersive fullscreen experience — and undoing it on form pages
- Toggling Android swipe-back/forward navigation gestures at runtime based on the current view
- Enabling pinch-to-zoom or pull-to-refresh — both are App Studio configuration (pinch-to-zoom is disabled by default), not bridge calls
- Locking orientation (portrait/landscape) — must be done in App Studio configuration, not JS
- Forcing light/dark scheme for native UI menus and driving web dark mode via `prefers-color-scheme`

Don't use for:

- Keyboard visibility/size tracking (used alongside fullscreen to fix form breakage) — see the `median-device-apis` skill for `median.keyboard.*`
- Device info, clipboard, share, downloads — `median-device-apis`
- Plugin-gated features (haptics, contacts, datastore, modals, review) — `median-native-features`

## Prerequisites

- **JavaScript Bridge enabled** in App Studio (required for every bridge call).
- No additional native plugins required for any API in this skill.
- Defaults settable in App configuration on the **Interface** tab: keep-screen-on default mode, Android fullscreen default mode, and iOS landscape fullscreen (hiding sidebars) — all configured there, not via the bridge.
- Orientation modes are configured in App Studio (App configuration) per OS and device type — see Orientation section below.
- Swipe navigation can be toggled at runtime via the **Android-scoped** bridge command (`median.android.swipeGestures.*`); **pinch-to-zoom** (disabled by default) and **pull-to-refresh** are configured per-platform in App Studio, not via the bridge.

## Quick Reference

| Task | Exact call |
|---|---|
| Set brightness to 80% | `median.screen.setBrightness({'brightness':'0.8'});` |
| Set brightness, restore on navigation | `median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true});` |
| Reset brightness to system default | `median.screen.setBrightness({'brightness':'default'});` |
| Keep screen awake | `median.screen.keepScreenOn();` |
| Return to normal sleep behavior | `median.screen.keepScreenNormal();` |
| Android: enter fullscreen | `median.android.screen.fullScreen();` |
| Android: exit fullscreen | `median.android.screen.normal();` |
| Android: enable swipe navigation | `median.android.swipeGestures.enable();` |
| Android: disable swipe navigation | `median.android.swipeGestures.disable();` |
| Pinch-to-zoom / pull-to-refresh | **No bridge command** — App Studio configuration (pinch-to-zoom disabled by default) |
| Lock orientation | **No bridge command** — App Studio configuration only (Auto-Rotation / Fixed Portrait / Fixed Landscape) |
| Force dark scheme | `median.screen.setColorScheme("dark");` |
| Force light scheme | `median.screen.setColorScheme("light");` |
| Follow device setting | `median.screen.setColorScheme("auto");` |
| Revert scheme to app-config default | `median.screen.resetColorScheme();` |

## How It Works

### Call forms: library, NPM, protocol

- **Library (default):** `median.screen.*` — injected into the page inside the app.
- **NPM:** `import Median from "median-js-bridge";` then the same call shape with a capitalized `Median` (e.g. `Median.screen.keepScreenOn()`). Enable *Website Overrides > JavaScript Frameworks and NPM* in App Studio and disable library injection.
- **Protocol:** `median://<module>/<function>` via an anchor `href` or `window.location.href`, with parameters `encodeURIComponent()`-encoded; chain commands with `median://nativebridge/multi` (an array of URLs) instead of sequential calls.

See the `median-bridge-setup` skill for bridge readiness (`median_library_ready()`) and app detection.

### Screen brightness — `median.screen.setBrightness`

```javascript
median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true});
median.screen.setBrightness({'brightness':'default'});
```

- `brightness`: string `'0'` to `'1.0'` (e.g. `'0.8'` = 80%) or `'default'`.
- `restoreOnNavigation`: boolean — reverts to the previous setting after page navigation.
- Pass values as strings, per the docs examples.

**Doc typo warning:** the official docs' code snippets contain a stray trailing `'` after the closing paren (e.g. `...})';`). That is a typo in the docs — the call itself ends with `})`. Write it as shown above.

### Keep screen on — `median.screen.keepScreenOn` / `keepScreenNormal`

```javascript
median.screen.keepScreenOn();
median.screen.keepScreenNormal();
```

No parameters, no callback. A runtime wake lock: call `keepScreenOn()` when playback or an unattended flow starts, `keepScreenNormal()` when it ends. The default mode can be set in app configuration on the **Interface** tab.

### Fullscreen (Android) — `median.android.screen.fullScreen` / `normal`

```javascript
median.android.screen.fullScreen();
median.android.screen.normal();
```

- The JS command is **Android-scoped** — it hides Android navigation/status bars.
- iOS full screen in landscape (hiding sidebars) is configured via the **Interface** tab, not the bridge. The Android default mode is also settable on the Interface tab.

**Keyboard breakage (docs warning):** with Full Screen enabled on Android, the keyboard **overlays** web content and can break forms. Fixes: (1) disable full screen via the bridge on form pages, or (2) use the keyboard listener functions (`median.keyboard.listen`, see `median-device-apis`) to toggle full screen when the keyboard shows/hides:

```javascript
function keyboardToggle(data) {
  if (data.visible) { median.android.screen.normal(); }     // keyboard shown: exit fullscreen so it doesn't overlay
  else { median.android.screen.fullScreen(); }              // keyboard hidden: re-enter fullscreen
}
median.keyboard.listen(keyboardToggle);
```

### Swipe gestures (Android) — `median.android.swipeGestures.enable` / `disable`

```javascript
median.android.swipeGestures.enable();
median.android.swipeGestures.disable();
```

- Swipe gestures let users navigate **backward and forward through their browsing history** with horizontal swipes; toggle them at runtime to dynamically enable or disable navigation based on your app's current state or specific view requirements.
- On Android, **visual arrow indicators** appear on screen when a previous or next page is available in the history stack.
- Related gesture settings that are App Studio **configuration only** (no bridge command): **Pinch-to-Zoom** — disabled by default; enable for content that needs fine-grained zooming (product images, maps, data-dense pages) — and **Pull-to-Refresh** — configurable independently for iOS and Android.

### Screen orientation — config-only, NO bridge command

There is **no bridge command** to lock or change orientation at runtime. Screen orientation is controlled exclusively in **App Studio app configuration**, with modes: **Auto-Rotation**, **Fixed Portrait**, **Fixed Landscape** — customizable per Operating System (iOS/Android) and Device Type (Phone/Tablet).

**Gotcha:** enabling Fixed Portrait on iPads **automatically disables multi-tasking capabilities**.

If a request says "rotate the screen from JavaScript" — the correct answer is to set the orientation mode in app configuration and rebuild; JS cannot do it via the bridge.

### Dark mode — `median.screen.setColorScheme` / `resetColorScheme`

```javascript
if (navigator.userAgent.indexOf("median") > -1) {
  median.screen.setColorScheme("dark");
  median.screen.resetColorScheme();
}
```

- Takes a single string argument `"light" | "dark" | "auto"` — **not an options object**, per the docs examples.
- Affects native UI (nav/tab/sidebar menus) and drives web content: the app sets `prefers-color-scheme` to `light`/`dark` dynamically with device mode.
- Median also sets a `data-color-scheme-option` property on the page with the current app setting (`light`, `dark`, or `auto`) — use it to build a scheme toggle.
- The docs wrap calls in a `navigator.userAgent.indexOf("median") > -1` guard so the code no-ops outside the app.

Recommend CSS variables for light/dark palettes so one `prefers-color-scheme` switch re-themes the whole app.

## Pitfalls

- **Brightness values are strings.** Pass `'0.8'`, not `0.8`. `'default'` restores the system setting.
- **Don't copy the docs' stray quote.** Snippets on docs.median.co end `})';` — that trailing `'` is a doc typo; your call should end `})`.
- **Full screen + forms on Android.** The keyboard overlays content in fullscreen mode and can break form inputs — exit fullscreen (`normal()`) on form pages or toggle it from the keyboard listener.
- **iOS fullscreen is not bridge-controlled.** `median.android.screen.*` is Android-only; iOS landscape fullscreen is an Interface-tab setting.
- **Orientation cannot be changed from JS.** No bridge command exists — App Studio configuration only, and Fixed Portrait on iPad kills multi-tasking.
- **Swipe-gesture toggling is Android-scoped.** The documented runtime commands are `median.android.swipeGestures.enable()` / `.disable()`; there is no documented iOS equivalent, and pinch-to-zoom / pull-to-refresh have no bridge commands — App Studio configuration only.
- **`setColorScheme` takes a bare string.** `median.screen.setColorScheme("dark")`, not `setColorScheme({ scheme: "dark" })`.
- **No callbacks or promises documented** for any screen API in this skill — don't `.then()` or `await` them.
- **Outside the app there is no `median` object.** Guard with the UA check or `Median.isNativeApp()` (NPM) so the same code runs on the mobile web.

## Verification

1. Brightness: `setBrightness({'brightness':'0.2'})` visibly dims the screen; `'default'` restores the system level; with `restoreOnNavigation: true` the previous level returns after a page navigation.
2. Keep-awake: call `keepScreenOn()`, leave the device idle past its normal lock timeout — the screen stays on; `keepScreenNormal()` restores normal sleep behavior.
3. Fullscreen: `fullScreen()` hides the Android status/navigation bars and `normal()` restores them; with the keyboard-listener toggle wired up, exiting fullscreen when the keyboard shows keeps form inputs visible.
4. Dark mode: `setColorScheme("dark")` flips the native menus and the page's `prefers-color-scheme`; the page's `data-color-scheme-option` reflects the mode; `resetColorScheme()` returns to the app-config default.
5. Swipe gestures (Android): after `enable()`, swiping horizontally navigates back/forward through history and the arrow indicators appear when a previous/next page exists; after `disable()`, the same swipes no longer navigate.

## References

- [references/brightness-keepawake-fullscreen.md](references/brightness-keepawake-fullscreen.md) — brightness params + typo note, keepScreenOn, Android fullscreen + keyboard fix
- [references/orientation-dark-mode.md](references/orientation-dark-mode.md) — orientation config modes, dark-mode scheme calls, `prefers-color-scheme` and `data-color-scheme-option`
- [Repo recipes](../../recipes.md) — keyboard-aware forms on Android (keyboard listener + fullscreen toggle)
- Official docs: [Screen Brightness](https://docs.median.co/docs/screen-brightness) · [Keep Screen On](https://docs.median.co/docs/keep-screen-on) · [Full Screen](https://docs.median.co/docs/full-screen) · [Screen Orientation](https://docs.median.co/docs/screen-orientation) · [Dark Mode](https://docs.median.co/docs/dark-mode) · [Native Gestures and Interactions](https://docs.median.co/docs/swipe-gestures)
- Live demo pages: [Swipe Gestures](https://median.dev/swipe-gestures/)
