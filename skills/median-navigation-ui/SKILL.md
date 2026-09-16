---
name: median-navigation-ui
description: "Drive native navigation and link routing in a Median.co app from JS — tab bar, sidebar, dynamic top-bar titles, status bar, Auto New Windows multi-level navigation, link-handling rules (internal/external/appbrowser), and the long-press context menu. Use when wiring native nav menus, controlling where links open, or styling navigation chrome."
version: 0.2.0
author: Dennis Rongo (dennisrongo), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Median, JavaScript Bridge, Mobile, WebView]
  source: https://docs.median.co/docs/native-navigation-overview
---

# Median Navigation & UI

Median apps render native navigation — a top navigation bar, a slide-out sidebar, a bottom tab bar, and (iOS only) a contextual navigation toolbar — that draws instantly, before any web content loads, and persists across internal and third-party pages. This skill drives all of them at runtime from JS (`median.tabNavigation.*`, `median.sidebar.*`, `median.navigationTitles.*`, `median.ios.contextualNavToolbar.*`) plus native status-bar styling via `median.statusbar.set`, multi-level "< Back" navigation via `median.navigationLevels.set` (Auto New Windows), link routing via `median.window.open` / App Studio Link Behavior rules / `median.internalExternal.set`, and the long-press context menu via `median.contextMenu.*`.

## When to Use

Use this skill when:

- Defining, replacing, or hiding the bottom tab bar from the website (`median.tabNavigation.setTabs`)
- Programmatically selecting/deselecting tabs (`selectTab`/`deselect`) or highlighting the right tab for SPA route changes
- Setting sidebar menu items at runtime (`median.sidebar.setItems`), including group headers and `javascript:` links
- Setting per-page top-bar titles dynamically (`navigationTitles.set`/`setCurrent`)
- Showing/hiding the iOS contextual navigation toolbar (back/forward/refresh)
- Styling the status bar at runtime (style, color, overlay, blur) or auto-matching it to the page background
- Building multi-level navigation where matched pages open in a new native window with a "< Back" button (`median.navigationLevels.set` — Auto New Windows)
- Controlling where links open — programmatically (`median.window.open` in `blank`/`internal`/`external`/`appbrowser` mode) or by rule (App Studio Link Behavior, runtime `median.internalExternal.set`)
- Detecting when the user closes the in-app browser (`median_appbrowser_closed()` hook)
- Enabling the long-press context menu on links and choosing its actions (`median.contextMenu.setEnabled` / `setActions`)

Don't use for:

- Setting up the bridge itself (injected library vs NPM vs protocol) — see `median-bridge-setup`
- Webview zoom/reload/cache or keyboard state — see `median-device-apis`
- Brightness, orientation, dark mode, or swipe gestures — see `median-screen-controls`
- Other native capabilities (media, scanning, push/auth, analytics/IAP) — see the sibling skills `median-native-features`, `median-scanning`, `median-push-auth`, `median-analytics-iap`

## Prerequisites

- A Median app on a plan tier that includes native navigation (Top Navigation Bar, Sidebar Navigation, Bottom Tab Bar) and the features you invoke.
- Navigation components are configured at build time in App Studio (Native Navigation tab) and/or at runtime via the JS Bridge calls below. Auto New Windows levels and Link Behavior rules follow the same pattern — App Studio config that the website can override dynamically.
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
| Set Auto New Windows levels | `median.navigationLevels.set({'active':true,'persist':true,'levels':[{regex:'.*median.*',level:2},{regex:'.*',level:1}]});` |
| Revert levels to app config | `median.navigationLevels.set({'persist': true});` |
| Open URL in a specific mode | `median.window.open(url, mode);` — `'blank'` (default) \| `'internal'` \| `'external'` \| `'appbrowser'` |
| Override link rules at runtime | `median.internalExternal.set({rules:[{id:1,regex:'https?://maps\\.google\\.com.*',mode:'external'}]});` |
| Detect in-app browser close | define `function median_appbrowser_closed() {...}` on the page — called automatically on close |
| Enable/disable long-press menu | `median.contextMenu.setEnabled(true);` / `median.contextMenu.setEnabled(false);` |
| Set context-menu actions | `median.contextMenu.setActions(["copyLink","openExternal"]);` |
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

### Auto New Windows — `median.navigationLevels.set`

Pages matched to a higher `level` rule open in a new native window (Android Activity / iOS ViewController) with a "< Back" button in the top navigation bar — the news/banking-app style of hierarchical navigation. Rules are prioritized top to bottom; a link matching no rule opens in the current level. Configure at build time (App Studio / appConfig.json) or at runtime:

```javascript
var autoNewWindowsRules = {
    active: true,
    persist: true,
    levels: [{
      regex: '.*median.*',
      level: 2
    }, {
      regex: '.*',
      level: 1
    }]
}
median.navigationLevels.set(autoNewWindowsRules);
median.navigationLevels.set({persist: true}); // revert to appConfig.json
```

`persist: true` saves the navigation levels for the next app launch; otherwise they apply to the current session only. Two documented caveats:

- **New windows require a full page load** — the page loads in a new webview. In an AJAX single-page app, force the load with `window.location.href = 'https://my-ajax-site/path";` *(docs snippet shows mismatched quotes — preserved verbatim)*.
- **No history inheritance** — a new window does not carry the prior page's navigation history; swipe gestures only take the user back to that window's originally opened URL.

Full demo configuration JSON and set/revert snippets: [references/navigation-api.md](references/navigation-api.md).

### Link Handling — `median.window.open` + Link Behavior rules

Open a URL programmatically in one of four modes:

```javascript
median.window.open(url, mode);
// mode: 'blank' (default — new WebView window) | 'internal' (current WebView)
//     | 'external' (system browser or registered deep-link app)
//     | 'appbrowser' (in-app browser window)
```

Regular page links follow the **Link Behavior rules** (App Studio): by default, same-domain links open **internal** and other domains open in the **app browser** (the "All Other Links" rule — editable to Internal or External). Rules evaluate **top to bottom**, so put the most specific first. Match types: Single Page, Multiple Pages (URL path prefix), All Pages, Custom (your own regex), each with an Include Subdomains toggle.

**Regex compatibility**: lookahead/lookbehind expressions (including negative variants) are unsupported on Android and only available on iOS 14+ — such rules silently fail to match. Use simple, positive patterns.

Override the rules at runtime:

```javascript
var rulesArray = [
  { id: 1, regex: "https?://maps\\.google\\.com.*", mode: "external" },
  { id: 4, regex: "https?://([-\\w]+\\.)*nytimes\\.com/.*", mode: "appbrowser" }
];
median.internalExternal.set({ rules: rulesArray });
```

To detect when the user closes the in-app browser, define `median_appbrowser_closed()` on the page — the app calls it automatically on close. On iOS, the in-app browser's native control colors come from App Studio → Branding. Full rule table and complete examples: [references/navigation-api.md](references/navigation-api.md).

### Long-Press Context Menu — `median.contextMenu.*`

The context menu is a floating menu shown when the user long-presses a link. Turn it on or off at runtime and choose which actions appear:

```javascript
median.contextMenu.setEnabled(true); // or false to disable
// Define which actions appear in the context menu
median.contextMenu.setActions(["copyLink", "openExternal"]);
```

`copyLink` copies the selected link to the clipboard; `openExternal` opens it in the device's external browser.

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

### Recent platform updates

Two build-time options were added to App Studio on Feb 10, 2026 ([App Studio Release Notes](https://docs.median.co/docs/release-notes-app-studio)): an option to **hide the top and bottom navigation bars on scroll** (in the Top Navigation Bar and Bottom Tab Bar tabs under Native Navigation), and an option to enable **Liquid Glass** on iOS (in the Interface tab). Liquid Glass is Apple's iOS 26 frosted-glass design effect — Median apps support it automatically on compatible iOS devices once "Modern Liquid Design" is turned on, and it integrates natively with web content using CSS `backdrop-filter` ([Liquid Glass docs](https://docs.median.co/docs/ios-liquid-glass)).

## Pitfalls

- **`selectTab` is 0-indexed** — `selectTab(1)` selects the *second* tab, not the first.
- **`setCurrent` title must be URL-encoded** — `{'title':'Hello%20World'}`; a raw `'Hello World'` is not the documented form.
- **`sidebar.setItems` needs `enabled: true`** to activate a hidden sidebar — items alone won't show it.
- **SPA route changes don't re-run page-load code** — update tab selection (`selectTab`) on route change, or use per-tab `regex` rules so the active tab follows the URL.
- **Conflicting active tabs** — use specific regexes per tab; overlapping patterns can light up multiple tabs.
- **`persist: true` state survives app restarts** — tab menus, sidebar items, titles, and navigation levels saved with `persist` reload on every subsequent launch. Revert deliberately (e.g. `median.navigationTitles.set({'persist': true})`, `median.navigationLevels.set({'persist': true})`), not by accident.
- **Auto New Windows requires a full page load** — an AJAX SPA route change will not open a new window; force it with `window.location.href`.
- **New windows don't inherit history** — swipe-back inside a new window returns only to that window's first URL, not to the page that opened it.
- **Regex lookaround silently fails** — lookahead/lookbehind in link rules and URL regexes is unsupported on Android and iOS < 14; the rule just fails to match with no error. Use simple positive patterns.
- **Default link routing surprises** — same-domain links open internal, everything else defaults to the app browser; change the "All Other Links" rule in App Studio or override at runtime with `median.internalExternal.set` before assuming links are "broken".
- **Contextual toolbar is iOS-only** — `median.ios.contextualNavToolbar.set` has no Android equivalent; don't gate cross-platform navigation on it.
- **Android `overlay: true` keyboard bug** — the keyboard overlays web content, breaking form completion. Disable overlay on form pages or toggle it via keyboard listener functions.
- **`blur` is iOS-only** — ignored on Android.
- **Test on both platforms** — iOS follows Apple HIG, Android follows Material Design; tab/sidebar rendering differs.
- **Icons** — use Font Awesome / Material Design / custom SVG classes (e.g. `"fas fa-cog"`); separate active/inactive tab icons are supported at build time.
- **Top bar visibility** — the Top Navigation Bar only appears if the app uses Sidebar Navigation, Auto New Windows, Search, Refresh Button, or Custom Buttons; titles set via the Bridge won't show without one of those.
- **Legacy GoNative apps** — substitute the `gonative.` prefix (e.g. `gonative.navigationTitles.setCurrent(...)`); the docs' own anchor examples use it, and both prefixes work on Median-updated apps.

## Verification

1. Tab select/deselect reflects immediately in-app; a URL-rule (`regex`) tab highlights on a deep-linked page load.
2. `setCurrent` title shows in the top bar on the target page; `navigationTitles.set` rules survive an app restart (`persist: true`), and `set({'persist': true})` reverts to the build-time titles.
3. Auto New Windows: a link matching a higher `level` opens a new native window with a "< Back" button; after `median.navigationLevels.set({'persist': true})` and an app restart, the appConfig.json levels are back in effect.
4. SPA caveat confirmed: an AJAX route change does NOT open a new window; forcing `window.location.href` to a higher-level URL does.
5. Link handling: a same-domain link opens internal; a link matched by an `external` rule leaves the app for the system browser; an `appbrowser` link opens the in-app browser; the defined `median_appbrowser_closed()` fires when it closes.
6. Runtime rules: after `median.internalExternal.set({...})`, a page matching the new regex routes to the configured mode with no app rebuild.
7. Long-press a link with the context menu enabled — `copyLink` puts the URL on the clipboard and `openExternal` opens the device browser, on both iOS and Android.
8. Regex audit: no link-rule, tab, or navigation-level regex uses lookahead/lookbehind (they silently fail on Android and iOS < 14).

## References

- [references/navigation-api.md](references/navigation-api.md) — full verbatim parameter tables, tab-selection regex JSON, `toolbarNavigation` build-time config, Auto New Windows demo configuration, link-handling rules and runtime overrides, context menu, all doc snippets
- [Repo recipes](../../recipes.md) — cross-skill combos featuring this skill
- Official docs: [Native Navigation Overview](https://docs.median.co/docs/native-navigation-overview) · [Top Navigation Bar](https://docs.median.co/docs/top-navigation-bar) · [Sidebar Navigation Menu](https://docs.median.co/docs/sidebar-navigation-menu) · [Bottom Tab Bar](https://docs.median.co/docs/bottom-tab-bar) · [Selecting Tabs](https://docs.median.co/docs/selecting-tabs) · [Dynamic Menu Items](https://docs.median.co/docs/dynamic-menu-items) · [Dynamic Tab Menu](https://docs.median.co/docs/dynamic-tab-menu) · [Dynamic Titles](https://docs.median.co/docs/dynamic-titles) · [Auto New Windows](https://docs.median.co/docs/auto-new-windows) · [Link Handling Overview](https://docs.median.co/docs/link-handling-overview) · [Link Behavior](https://docs.median.co/docs/internal-vs-external-links) · [Contextual Navigation Toolbar](https://docs.median.co/docs/contextual-navigation-toolbar) · [Status Bar](https://docs.median.co/docs/status-bar) · [Liquid Glass (iOS)](https://docs.median.co/docs/ios-liquid-glass) · [App Studio Release Notes](https://docs.median.co/docs/release-notes-app-studio)
- Live demo pages (open inside your app): https://median.dev/tab-bar-navigation/ · https://median.dev/sidebar-navigation/ · https://median.dev/top-navigation-bar/ · https://median.dev/status-bar/ · https://median.dev/auto-new-windows/ · https://median.dev/link-handling/ · https://median.dev/context-menu/
