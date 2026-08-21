# Median Navigation & UI — Full API Reference

Verbatim API detail from docs.median.co (fetched 2026-08-20). Nothing invented; snippets reproduced exactly from the docs.

---

## Native Navigation Overview

- Native navigation menus render instantly (before web content loads) and persist across internal and third-party pages — built at build time in App Studio or at runtime via the JS Bridge.
- Four component types: **Top Navigation Bar**, **Sidebar Navigation**, **Bottom Tab Bar**, and (iOS only) **Contextual Navigation Toolbar**.
- Benefits (docs): flexible UI control (build-time or runtime via Bridge), cross-service navigation via deep links + native tabs/sidebars, persistent navigation across internal/external pages, immediate rendering before any web content loads.
- Icon support: standard + custom libraries (Font Awesome, Material Design, custom SVG); separate active/inactive tab icons. See Custom Icons docs.
- Platform guidelines: iOS — Apple HIG; Android — Material Design.

Source: https://docs.median.co/docs/native-navigation-overview

---

## Top Navigation Bar

Only visible if using one of: Sidebar Navigation, Auto New Windows, Search, Refresh Button, or Custom Buttons.

Set per-page title (verbatim):

```html
<script>
  function median_library_ready() {
    median.navigationTitles.setCurrent({ title: "Your Title" });
  }
</script>
```

Developer demo: https://median.dev/top-navigation-bar/
Source: https://docs.median.co/docs/top-navigation-bar

---

## Bottom Tab Bar

iOS implementation follows Apple HIG (tab bars); Android follows Material Design (bottom navigation). Dynamic control via `median.tabNavigation.*` (see below).

Developer demo: https://median.dev/tab-bar-navigation/
Source: https://docs.median.co/docs/bottom-tab-bar

---

## median.tabNavigation.setTabs (dynamic tab menu)

Define/replace bottom tab bar buttons at runtime, or toggle tab menu visibility. A default tab menu can be defined in app config and overwritten dynamically; config can be left blank with all tab menus set by the website.

Call signatures:

- Set/change tabs: `median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});`
- Hide tab menu: `median.tabNavigation.setTabs({'enabled': false});`

Tab item shape: `icon` (optional, e.g. `"fas fa-cloud"`), `label`, `url` (can be `javascript:` URI).

Verbatim example:

```javascript
var tabItems = [{
    "icon": "fas fa-cloud", //optional
    "label": "Tab 1",
    "url": "javascript:alert('You selected tab 1')"
}, {
    "icon": "fas fa-globe", //optional
    "label": "Tab 2",
    "url": "javascript:alert('You selected tab 2')"
}, {
    "icon": "fas fa-users", //optional
    "label": "Tab 3",
    "url": "javascript:alert('You selected tab 3')"
}];
median.tabNavigation.setTabs({'enabled': true, 'items': tabItems});
```

Source: https://docs.median.co/docs/dynamic-tab-menu

---

## median.tabNavigation.selectTab / deselect

- `median.tabNavigation.selectTab(1);` — tabs are **0-indexed**; `selectTab(1)` selects the *second* tab.
- `median.tabNavigation.deselect();` — deselect all tabs.

When needed: navigation from link/push/redirect onto a tab page; custom web navigation alongside the tab bar; highlighting the correct tab for dynamic routes, nested pages, or SPA views.

### URL-rule alternative — per-tab `regex`

Add a `regex` field per tab item in the tab-menu JSON; the tab shows active whenever a matching page is displayed. Docs example (note `"active": true` at root and `tabSelectionConfig` mapping):

```json
{
  "tabSelectionConfig": [ { "id": "1", "regex": ".*" } ],
  "tabMenus": [
    {
      "id": "1",
      "items": [
        {
          "subLinks": [],
          "label": "My Account",
          "icon": "fas fa-user",
          "url": "https://domain.com/account",
          "regex": "https://domain\\.domain/account.*"
        }
      ]
    }
  ],
  "active": true
}
```

Best practices (docs): JS Bridge selection when the site controls nav state; URL rules when the active tab should follow the page URL; use specific regexes per tab to avoid conflicting active states; test on both iOS and Android; in SPAs, update tab selection on route change since no full page load occurs.

Source: https://docs.median.co/docs/selecting-tabs

---

## median.sidebar.setItems (dynamic menu items)

Set sidebar navigation menu options at runtime.

Call signature: `median.sidebar.setItems({"items":items,"enabled":true,"persist":true});`

Parameters:

- `enabled` — **required to activate the sidebar** if it is hidden.
- `persist` — keeps the changes after the app is closed and reloaded.

Item shape: `label`, `url`, `icon` (optional, Font Awesome class e.g. `"fas fa-cog"`), `isGrouping: true` for a group header whose children live in `subLinks: [...]`, and `url` may be a `javascript:` URI.

Verbatim example:

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
    }, {
        label: "Google",
        url: "https://google.com",
        icon: "fas fa-home" //optional
    }]
}, {
    label: "Sample Javascript",
    url: "javascript:alert('test')"
}];
median.sidebar.setItems({"items":items,"enabled":true, "persist":true});
```

Developer demo: https://median.dev/sidebar-navigation/
Source: https://docs.median.co/docs/dynamic-menu-items

---

## median.navigationTitles.set / setCurrent (dynamic titles)

Configure Top Navigation Bar title text/images per URL — at build time (App Studio Dynamic Titles) or runtime (Bridge).

Build-time config: list of `{regex, title}` or `{regex, showImage: true}` rules; rules prioritized top-to-bottom; no match → app name shown.

### Runtime — set full title structure (verbatim)

```javascript
var menuItems = {
    active: true,
    titles: [{
        showImage: true,
        regex: '/home'
    },{
        title: 'my title',
        regex: '.*'
    }]
}
median.navigationTitles.set({
  'persist':true,
  'data': menuItems});
```

`persist=true` saves titles for the next app launch; otherwise current-session only.

### Runtime — revert to build-time config

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    median.navigationTitles.set({'persist':true});
}
```

### Runtime — one-off current page title (**URL-encode the title**)

```javascript
if (navigator.userAgent.indexOf('median') > -1) {
    median.navigationTitles.setCurrent({'title':'Hello%20World'})
}
```

```html
<a onclick="gonative.navigationTitles.setCurrent({'title':'Hello%20World'})">Set Current Page's Title</a>
```

Note: the anchor example in the docs uses the `gonative.` prefix — both prefixes work on Median-updated apps.

Source: https://docs.median.co/docs/dynamic-titles

---

## median.ios.contextualNavToolbar.set (iOS-only)

Native bottom toolbar with back (default), optional forward and refresh buttons — because iOS lacks a hardware back button.

**iOS-only feature; NOT supported on Android.**

Call signature (verbatim):

```javascript
// Show the contextual navigation toolbar, if conditions are met
median.ios.contextualNavToolbar.set({"enabled":true});
// Hide the contextual navigation toolbar
median.ios.contextualNavToolbar.set({"enabled":false});
```

Display conditions when `enabled: true` — toolbar appears if ANY of:

- webview has back history;
- forward button enabled AND webview has forward history;
- refresh button enabled.

`enabled: false` → always hidden.

### Build-time config — `toolbarNavigation` in `appConfig.json`

Key options:

- `visibilityByBackButton`: `"backButtonActive"` (default — show only when history exists) | `"always"`
- `visibilityByPages`: `"allPages"` | `"specificPages"` (with `regexes` array; visibility based on the **first matched regex**)
- `items[]`: each `{ "system": "back" | "refresh" | "forward", "enabled", "title", "titleType": "noText" | "defaultText" | "customText", "visibility": "allPages" | "specificPages", "urlRegex": [{ "enabled": true|false, "regex": ".*" }] }`

Default config (verbatim):

```json
"toolbarNavigation": {
  "enabled": true,
  "visibilityByPages": "allPages",
  "visibilityByBackButton": "backButtonActive",
  "regexes": [ { "enabled": true, "regex": ".*" } ],
  "items": [
    { "system": "back", "titleType": "defaultText", "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] },
    { "system": "refresh", "enabled": false, "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] },
    { "system": "forward", "enabled": false, "titleType": "defaultText", "visibility": "allPages",
      "urlRegex": [{ "enabled": true, "regex": ".*" }] }
  ]
}
```

Default appearance: Back button with text label (`< Back`); labels can be hidden or customized (`titleType: "customText"` + `title`).

Source: https://docs.median.co/docs/contextual-navigation-toolbar

---

## median.statusbar.set (status bar styling)

Set status bar style/visibility/color at runtime; configure Light/Dark/Auto mode.

Call signature (verbatim):

```javascript
// Dynamically
if (navigator.userAgent.indexOf('median') > -1) {
  median.statusbar.set({
    'style':'light',
    'color':'80ff0000',
    'overlay':true,
    'blur': true // optional - iOS only
  });
}

// Or on page load
function median_library_ready(){
  median.statusbar.set({
    'style':'light',
    'color':'80ff0000',
    'overlay':true,
    'blur': true // optional - iOS only
  });
}
```

### Parameters (verbatim from docs)

| Param | Type / values | Notes |
| --- | --- | --- |
| `style` | `'light'` \| `'dark'` \| `'auto'` | Text/icon colors. Light mode = black text, Dark mode = white text, Auto follows device Light/Dark mode setting. |
| `color` | `RRBBGG` or `AARRBBGG` hex | Solid status bar color. `'00000000'` = completely transparent. Example `'80ff0000'` = red at 50% alpha. |
| `overlay` | `true` \| `false` | `true` = web content extends underneath the status bar. `false` (default) = web content starts below the status bar. |
| `blur` | `true` \| `false` — **iOS only** | `true` applies a blur effect over the status bar color; `false` (default) keeps the color as specified by `color`. |

### Auto-match status bar to page background

Built-in helper, called after library ready:

```javascript
function median_library_ready(){
  median_match_statusbar_to_body_background_color();
}
```

### Android keyboard/overlay gotcha

With `overlay: true` on Android, the **keyboard overlays web content**, breaking form completion. Docs remedies:

1. Disable overlay via the Bridge on pages with forms;
2. Use keyboard listener functions to toggle overlay when the keyboard is active/inactive.

Developer demo: https://median.dev/status-bar/
Source: https://docs.median.co/docs/status-bar
