# Median.co JavaScript Bridge — Agent Skills

Agent skills that teach your AI coding agent (Claude Code, Codex, Cursor, OpenCode, Hermes, …) how to wire the [Median.co JavaScript Bridge](https://docs.median.co/docs/javascript-bridge) into an existing web app running inside a Median-built iOS/Android app.

Built as companion material for the CodingMenace Median.co video. The skills are distilled **verbatim from the official docs** — exact call signatures, parameters, callback shapes, and platform gotchas — so your agent writes bridge code that works the first time.

## What's inside

| Skill | Covers |
|---|---|
| `median-bridge-setup` | Library vs NPM package vs `median://` protocol, `median_library_ready()`, detecting app vs browser, callbacks in iframes, GTM |
| `median-navigation-ui` | Native navigation: top bar, sidebar, bottom tab bar, dynamic menus/titles, tab selection, status bar, contextual toolbar |
| `median-device-apis` | Device info, clipboard, share sheet, downloads, webview cache, app-resumed, keyboard state |
| `median-screen-controls` | Brightness, keep-screen-on, fullscreen, orientation, dark mode |
| `median-native-features` | Haptics, calendar, contacts, native datastore, reader/secure modals, app review prompt, share-into-app, web screenshot |
| `median-push-auth` | Push notifications (OneSignal + programmatic), user consent & tags, Face ID/biometrics, social login, Clerk/Auth0, passkeys |
| `median-analytics-iap` | Analytics plugins (Adjust, AppsFlyer, Firebase), attribution, in-app purchases, appConfig.json, deep linking, offline downloads |

## Install

> Install commands below are verified from a clean state.

Using the [vercel-labs `skills` CLI](https://github.com/vercel-labs/skills) (installs into every major coding agent at once):

```bash
npx -y skills add dennisrongo/median-co-bridge-skills -l        # list skills first
npx -y skills add dennisrongo/median-co-bridge-skills -s '*' -y # install all skills
```

Or grab a single skill manually — each `skills/<name>/SKILL.md` is self-contained.

## Usage

Skills are triggered automatically by your agent when you ask for the relevant work ("add push notifications to my Median app", "make the tab bar switch from JS"). Each skill contains:

- **Setup prerequisites** (app plan tier, plugin enablement in App Studio)
- **Exact bridge calls** — library, NPM (`Median.`), and `median://` protocol forms
- **Platform differences** (iOS vs Android) and license-tier gates
- **Copy-paste snippets** lifted from the official docs

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

## License

MIT — see [LICENSE](LICENSE).
