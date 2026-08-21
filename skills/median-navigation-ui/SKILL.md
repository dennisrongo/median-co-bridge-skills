---
name: median-navigation-ui
description: "Control native nav bars, tabs, and status bar from JS."
version: 0.1.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
---

# Median Navigation & UI

Median apps render native navigation — a top navigation bar, a slide-out sidebar, a bottom tab bar, and (iOS only) a contextual navigation toolbar — that draws instantly, before any web content loads, and persists across internal and third-party pages. This skill drives all of them at runtime from JS (`median.tabNavigation.*`, `median.sidebar.*`, `median.navigationTitles.*`, `median.ios.contextualNavToolbar.*`) plus native status-bar styling via `median.statusbar.set`.

## When to Use

Use this skill when:

- Defining, replacing, or hiding the bottom tab bar from the website (`median.tabNavigation.setTabs`)
- Programmatically selecting/deselecting tabs (`selectTab`/`deselect`) or highlighting the right tab for SPA route changes
- Setting sidebar menu items at runtime (`median.sidebar.setItems`), including group headers and `javascript:` links
- Setting per-page top-bar titles dynamically (`navigationTitles.set`/`setCurrent`)
- Showing/hiding the iOS contextual navigation toolbar (back/forward/refresh)
- Styling the status bar at runtime (style, color, overlay, blur) or auto-matching it to the page background

Don't use for:

- Setting up the bridge itself (injected library vs NPM vs protocol) — see `median-bridge-setup`
- Non-navigation UI (haptics, modals, brightness, orientation) — separate skills cover those

## Prerequisites

- A Median app on a plan tier that includes native navigation (Top Navigation Bar, Sidebar Navigation, Bottom Tab Bar) and the features you invoke.
- Navigation components are configured at build time in App Studio (Navigation tab) and/or at runtime via the JS Bridge calls below.
- No NPM package required — every call below uses the injected `median.*` library. If you use the NPM package instead, prefix with capitalized `Median.` (e.g. `Median.statusbar.set(...)`); the parameters are identical.

## Quick Reference

| Task | Exact call |
|---|---|
| Set/replace bottom tabs | `median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});` |
| Hide the tab bar | `median.tabNavigation.setTabs({'enabled': false});` |
| Select a tab (0-indexed) | `median.tabNavigation.selectTab(1);` — selects the **second** tab |
| Deselect all tabs | `median.tabNavigation.deselect();` |
| Set sidebar items | `median.sidebar.setItems({"items": items, "enabled": true, "persist": true});` |
| Set top-bar titles | `median.navigationTitles.set({'persist': true, 'data': menuItems});` |
| Revert to build-time titles | `median.navigationTitles.set({'persist': true});` |
| One-off current page title | `median.navigationTitles.setCurrent({'title':'Hello%20World'})` |
| Show/hide contextual toolbar (iOS) | `median.ios.contextualNavToolbar.set({"enabled":true})` / `({"enabled":false})` |
| Style the status bar | `median.statusbar.set({'style':'light','color':'80ff0000','overlay':true,'blur':true})` |
| Match status bar to page | `median_match_statusbar_to_body_background_color();` after library ready |

## How It Works

### Bottom Tab Bar — `median.tabNavigation.*`

Define or replace tab bar buttons at runtime. A default tab menu can be defined in the app config and overwritten dynamically; the config can also be left blank with all tab menus set by the website.

```javascript
var tabItems = [{
    "icon": "fas fa-cloud", //optional
    "label": "Tab 1",
    "url": "javascript:alert('You selected tab 1')"
}, {
    "icon": "fas fa-globe", //optional
    "label": "Tab 2",
    "url": "https://example.com/tab2"
}];
median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});
```

Select/deselect programmatically — **tabs are 0-indexed**, so `selectTab(1)` selects the *second* tab:

```javascript
median.tabNavigation.selectTab(1); // second tab
median.tabNavigation.deselect();   // deselect all
```

For SPAs and dynamic routes, the docs recommend updating tab selection on route change (no full page load occurs). Alternatively, a `regex` field per tab item in the tab-menu JSON keeps the active tab following the page URL — full example in [references/navigation-api.md](references/navigation-api.md).

### Sidebar — `median.sidebar.setItems`

Set sidebar menu options at runtime. `enabled` is required to activate the sidebar if it is hidden; `persist` keeps changes after the app is closed and reloaded.

```javascript
var items = [{
    label: "Google",
    url: "https://google.com",
    icon: "fas fa-cog" // optional Font Awesome icon
}, {
    label: "Sample Grouping",
    isGrouping: true,
    subLinks: [{
        label: "Apple",
        url: "https://apple.com",
        icon: "fas fa-home" // optional
    }]
}, {
    label: "Sample Javascript",
    url: "javascript:alert('test')"
}];
median.sidebar.setItems({"items": items, "enabled": true, "persist": true});
```

Item shape details (group headers via `isGrouping: true`, children in `subLinks`, `javascript:` URIs allowed) are in [references/navigation-api.md](references/navigation-api.md).

### Top Navigation Bar — `median.navigationTitles.*`

The top bar is only visible if the app uses Sidebar Navigation, Auto New Windows, Search, Refresh Button, or Custom Buttons. Set a per-page title on load:

```html
<script>
  function median_library_ready() {
    median.navigationTitles.setCurrent({ title: "Your Title" });
  }
</script>
```

For a full title structure (regex → title/image rules, `persist` across launches), use `median.navigationTitles.set()`; calling it with only `{'persist': true}` reverts to the build-time App Studio config. **One-off `setCurrent` titles must be URL-encoded** — `'Hello%20World'`, not `'Hello World'`. Full snippets: [references/navigation-api.md](references/navigation-api.md).

### iOS Contextual Navigation Toolbar — `median.ios.contextualNavToolbar.set`

A native bottom toolbar with back (default), optional forward and refresh buttons — iOS lacks a hardware back button. **iOS-only; NOT supported on Android.**

```javascript
// Show the contextual navigation toolbar, if conditions are met
median.ios.contextualNavToolbar.set({"enabled":true});
// Hide the contextual navigation toolbar
median.ios.contextualNavToolbar.set({"enabled":false});
```

With `enabled: true` the toolbar appears if ANY of: the webview has back history; the forward button is enabled AND the webview has forward history; the refresh button is enabled. `enabled: false` → always hidden. Build-time `toolbarNavigation` config (visibility modes, per-item options) is in [references/navigation-api.md](references/navigation-api.md).

### Status Bar — `median.statusbar.set`

```javascript
function median_library_ready(){
  median.statusbar.set({
    'style':'light',       // 'light' | 'dark' | 'auto'
    'color':'80ff0000',    // RRBBGG or AARRBBGG hex
    'overlay':true,        // web content extends under the status bar
    'blur': true           // optional — iOS only
  });
}
```

| Param | Values | Notes |
|---|---|---|
| `style` | `'light'` \| `'dark'` \| `'auto'` | Text/icon colors. Light = black text, Dark = white text, Auto follows device setting. |
| `color` | `RRBBGG` / `AARRBBGG` hex | Solid status bar color. `'00000000'` fully transparent; `'80ff0000'` = red at 50% alpha. |
| `overlay` | `true` \| `false` | `true` = content extends underneath; `false` (default) = content starts below. |
| `blur` | `true` \| `false` — **iOS only** | `true` applies blur over the status bar color; `false` (default) keeps `color` as specified. |

Auto-match the status bar to the page background (built-in helper, call after library ready):

```javascript
function median_library_ready(){
  median_match_statusbar_to_body_background_color();
}
```

## Pitfalls

- **`selectTab` is 0-indexed** — `selectTab(1)` selects the *second* tab, not the first.
- **`setCurrent` title must be URL-encoded** — `{'title':'Hello%20World'}`; a raw `'Hello World'` is not the documented form.
- **`sidebar.setItems` needs `enabled: true`** to activate a hidden sidebar — items alone won't show it.
- **SPA route changes don't re-run page-load code** — update tab selection (`selectTab`) on route change, or use per-tab `regex` rules so the active tab follows the URL.
- **Conflicting active tabs** — use specific regexes per tab; overlapping patterns can light up multiple tabs.
- **Contextual toolbar is iOS-only** — `median.ios.contextualNavToolbar.set` has no Android equivalent; don't gate cross-platform navigation on it.
- **Android `overlay: true` keyboard bug** — the keyboard overlays web content, breaking form completion. Disable overlay on form pages or toggle it via keyboard listener functions.
- **`blur` is iOS-only** — ignored on Android.
- **Test on both platforms** — iOS follows Apple HIG, Android follows Material Design; tab/sidebar rendering differs.
- **Icons** — use Font Awesome / Material Design / custom SVG classes (e.g. `"fas fa-cog"`); separate active/inactive tab icons are supported at build time.
- **Top bar visibility** — the Top Navigation Bar only appears if the app uses Sidebar Navigation, Auto New Windows, Search, Refresh Button, or Custom Buttons; titles set via the Bridge won't show without one of those.
- **Legacy GoNative apps** — substitute the `gonative.` prefix (e.g. `gonative.navigationTitles.setCurrent(...)`); the docs' own anchor examples use it, and both prefixes work on Median-updated apps.

## References

- [references/navigation-api.md](references/navigation-api.md) — full verbatim parameter tables, tab-selection regex JSON, `toolbarNavigation` build-time config, all doc snippets
- Official docs: [Native Navigation Overview](https://docs.median.co/docs/native-navigation-overview) · [Top Navigation Bar](https://docs.median.co/docs/top-navigation-bar) · [Sidebar Navigation Menu](https://docs.median.co/docs/sidebar-navigation-menu) · [Bottom Tab Bar](https://docs.median.co/docs/bottom-tab-bar) · [Selecting Tabs](https://docs.median.co/docs/selecting-tabs) · [Dynamic Menu Items](https://docs.median.co/docs/dynamic-menu-items) · [Dynamic Tab Menu](https://docs.median.co/docs/dynamic-tab-menu) · [Dynamic Titles](https://docs.median.co/docs/dynamic-titles) · [Contextual Navigation Toolbar](https://docs.median.co/docs/contextual-navigation-toolbar) · [Status Bar](https://docs.median.co/docs/status-bar)
- Live demo pages (open inside your app): https://median.dev/tab-bar-navigation/ · https://median.dev/sidebar-navigation/ · https://median.dev/top-navigation-bar/ · https://median.dev/status-bar/
