# Median.co JavaScript Bridge — Agent Skills

Agent skills that teach your AI coding agent (Claude Code, Codex, Cursor, OpenCode, Hermes, …) how to wire the [Median.co JavaScript Bridge](https://docs.median.co/docs/javascript-bridge) into an existing web app running inside a Median-built iOS/Android app.

Built as companion material for the CodingMenace Median.co video. The skills are distilled **verbatim from the official docs** — exact call signatures, parameters, callback shapes, and platform gotchas — so your agent writes bridge code that works the first time. This library supersedes [`dennisrongo/median-co-skills`](https://github.com/dennisrongo/median-co-skills) (archived); its coverage lives on here.

## What's inside

| Skill | Covers |
|---|---|
| `median-bridge-setup` | Library vs NPM package vs `median://` protocol, `median_library_ready()`, detecting app vs browser, callbacks in iframes, GTM, GoNative legacy (`gonative.*`) migration reference |
| `median-navigation-ui` | Native navigation: top bar, sidebar, bottom tab bar, dynamic menus/titles, tab selection, Auto New Windows, link-handling rules (internal/appbrowser/external), long-press context menu, status bar, contextual toolbar |
| `median-device-apis` | Device info + minimum-version enforcement, clipboard, share sheet, downloads, webview cache & zoom, geolocation, camera capture, app-resumed, keyboard state |
| `median-screen-controls` | Brightness, keep-screen-on, fullscreen, orientation, dark mode, Android swipe gestures |
| `median-native-features` | Haptics, calendar, contacts, native datastore, reader/secure modals, app review prompt, share-into-app, web screenshot, background media player, JW Player |
| `median-scanning` | QR/barcode scanner, document scanner, NFC tag reading |
| `median-push-auth` | Push notifications (OneSignal + programmatic), user consent & tags, FCM and Customer.io plugin pointers, Face ID/biometrics, social login, Clerk/Auth0, passkeys |
| `median-analytics-iap` | Analytics plugins (Adjust, AppsFlyer, Firebase + Crashlytics), attribution, in-app purchases, RevenueCat paywalls, appConfig.json, deep linking, offline downloads |

Cross-skill combos (offline scan→store→sync, version-gated push identity, keyboard-aware forms, …) live in [recipes.md](recipes.md).

## Install

> Flags below match vercel-labs `skills` CLI v1.5.26; the list command is verified live against this repo.

Using the [vercel-labs `skills` CLI](https://github.com/vercel-labs/skills) (installs into every major coding agent at once):

```bash
npx -y skills add dennisrongo/median-co-bridge-skills -l             # list skills first
npx -y skills add dennisrongo/median-co-bridge-skills -s '*' -y      # install all skills (project scope)
npx -y skills add dennisrongo/median-co-bridge-skills -s '*' -g -y   # install all skills globally (~/<agent>/skills)
```

> **If `npx skills` launches a different CLI** — some other globally installed package whose binary is also named `skills` — force the registry package:
>
> ```bash
> npx -y --package skills skills add dennisrongo/median-co-bridge-skills -s '*' -g -y
> ```

Or grab a single skill manually — each `skills/<name>/SKILL.md` is self-contained.

## Usage

Skills are triggered automatically by your agent when you ask for the relevant work ("add push notifications to my Median app", "make the tab bar switch from JS"). Each skill contains:

- **Setup prerequisites** (app plan tier, plugin enablement in App Studio)
- **Exact bridge calls** — library, NPM (`Median.`), and `median://` protocol forms
- **Platform differences** (iOS vs Android) and license-tier gates
- **Copy-paste snippets** lifted from the official docs
- A **Verification checklist** of end-to-end checks for the feature you just wired

## The 30-second version

```html
<script>
  function median_library_ready() {
    median.share.sharePage({ url: 'https://example.com', text: 'Check this out' });
  }
  if (window.median) median_library_ready();
</script>
```

For React/Vue/SPAs, use the [NPM package](skills/median-bridge-setup) instead — `npm install median-js-bridge` and toggle *App Studio → Website Overrides → JavaScript Frameworks and NPM*.

## Provenance

References were distilled from docs.median.co — append `.md` to any docs URL for its markdown source; [llms.txt](https://median.co/llms.txt) is the full index. The [`research/`](research/) folder carries the original fetch provenance (63 doc pages, fetched and stamped 2026-08-20); the September 2026 coverage merge and refresh was verified against the live docs. When the docs and this package disagree, trust the docs and open an issue.

## License

MIT — see [LICENSE](LICENSE).
