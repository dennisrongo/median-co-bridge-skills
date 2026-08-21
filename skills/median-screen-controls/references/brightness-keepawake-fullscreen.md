# Brightness, Keep Screen On, Full Screen (full reference)

## median.screen.setBrightness (Screen Brightness)

Source: https://docs.median.co/docs/screen-brightness

**Purpose**: Set screen brightness programmatically (0–100% via 0–1.0), optionally restoring on navigation.

### Call signatures (verbatim from docs)

```javascript
median.screen.setBrightness({'brightness':'0.8'})
median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true})
median.screen.setBrightness({'brightness':'default'})
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `brightness` | string | `'0'` to `'1.0'` (e.g. `'0.8'` = 80%) or `'default'` |
| `restoreOnNavigation` | boolean | Reverts to previous setting after page navigation |

### Platform notes

None stated.

### Gotchas

- **Docs typo**: the docs' snippets contain a stray trailing `'` after the closing paren (e.g. `...})';`). The call itself ends `})`. Do not reproduce the stray quote.
- Pass values as strings per the docs examples.

### Minimal snippet

```javascript
median.screen.setBrightness({'brightness':'0.8', 'restoreOnNavigation': true});
median.screen.setBrightness({'brightness':'default'});
```

---

## median.screen.keepScreenOn / keepScreenNormal (Keep Screen On)

Source: https://docs.median.co/docs/keep-screen-on

**Purpose**: Prevent the screen from sleeping at runtime (wake lock).

### Call signatures

```javascript
median.screen.keepScreenOn()    // enable
median.screen.keepScreenNormal() // disable
```

### Parameters

None.

### Platform notes

- Default mode can be set in app configuration on the **Interface** tab.

### Minimal snippet

```javascript
median.screen.keepScreenOn();
median.screen.keepScreenNormal();
```

---

## median.android.screen.fullScreen / normal (Full Screen)

Source: https://docs.median.co/docs/full-screen

**Purpose**: Hide Android navigation/status bars (or iOS landscape sidebars) for a full-screen experience.

### Call signatures

```javascript
median.android.screen.fullScreen() // enable
median.android.screen.normal()     // disable
```

### Parameters

None.

### Platform notes

- JS command is **Android-scoped**.
- iOS full screen in landscape (hiding sidebars) is configured via the **Interface** tab, not the bridge.
- Default mode also settable on Interface tab for Android.

### Gotchas (docs warning)

With Full Screen enabled on Android, the keyboard **overlays** web content and can break forms. Fixes:

1. Disable full screen via the bridge on form pages, or
2. Use the keyboard listener functions to toggle full screen when the keyboard shows/hides:

```javascript
function keyboardToggle(data) {
  if (data.visible) { median.android.screen.normal(); }
  else { median.android.screen.fullScreen(); }
}
median.keyboard.listen(keyboardToggle);
```

### Minimal snippet

```javascript
median.android.screen.fullScreen();
median.android.screen.normal();
```
