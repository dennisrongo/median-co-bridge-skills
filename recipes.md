# Median Skill Recipes

Cross-skill recipes — each combines bridge APIs owned by different skills in this repo; the owning skills are listed under each heading.

## Offline capture pattern: scan → store → sync

**Skills:** [median-scanning](skills/median-scanning/SKILL.md) (barcode scan) · [median-native-features](skills/median-native-features/SKILL.md) (app storage) · [median-device-apis](skills/median-device-apis/SKILL.md) (`webview.reload`)

Median plugins that process data locally work without a network connection.
Recipe: launch the QR scanner while offline, persist the scanned value with
App Storage (not Cloud Storage — cloud is unavailable offline), then sync
when back online. Works as a custom offline.html page (self-contained assets
only: inline CSS/JS, base64/SVG images).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Offline Scan & Sync</title>
</head>
<body>
  <button onclick="scan()">Open QR Code Scanner</button>
  <input type="text" id="value" placeholder="Scanned Value" />
  <button onclick="save()">Save Value Locally</button>
  <button onclick="window.location.href='https://yourapp.com/'">Go Online</button>

  <script>
    const KEY = 'pendingScans';

    async function scan() {
      median.barcode.scan().then(function (data) {
        if (data.success) {
          document.getElementById("value").value = data.code;
        }
      });
    }

    function save() {
      median.storage.app.set({
        key: KEY,
        value: document.getElementById("value").value,
        statuscallback: function (result) {
          if (result.status) console.log(result.status);
        },
      });
    }
  </script>
</body>
</html>
```

On the online side, read the key back with `median.storage.app.get({ key })`
and POST to your backend, then `delete` the key.

Tip: set the app's `initialURL` to `file:///offline.html` for kiosk-style
offline-first launches. Manual retry button on the offline page:
`median.webview.reload();`

*(note: `median.webview.reload()` is not in the official docs — see the caveat in the `median-device-apis` skill; a `window.location.reload()` fallback also works inside the webview)*

## Version gate + push identity on login

**Skills:** [median-device-apis](skills/median-device-apis/SKILL.md) (`deviceInfo` version gate) · [median-push-auth](skills/median-push-auth/SKILL.md) (OneSignal identity capture)

Combine device-info version enforcement with OneSignal identity capture on
the same login flow:

```javascript
async function afterLogin(user) {
  // 1. gate old app versions before anything else
  const info = await median.deviceInfo();
  if (isVersionLower(info.appVersion, MIN_APP_VERSION)) {
    window.location.replace('/update-required.html');
    return;
  }

  // 2. associate push identity
  await median.onesignal.login(user.id);
  const os = await median.onesignal.info();

  // 3. persist for backend targeting
  await fetch('/api/users/' + user.id + '/push', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      oneSignalId: os.oneSignalId,
      subscriptionId: os.subscription.id,
      platform: info.platform,
      appVersion: info.appVersion,
    }),
  });
}
```

## Keyboard-aware forms on Android

**Skills:** [median-device-apis](skills/median-device-apis/SKILL.md) (keyboard listener) · [median-screen-controls](skills/median-screen-controls/SKILL.md) (fullscreen toggle) · [median-navigation-ui](skills/median-navigation-ui/SKILL.md) (status-bar overlay)

Full Screen and status-bar overlay both cause the keyboard to sit on top of
web content. Combine the keyboard listener with the screen controls:

```javascript
median.keyboard.listen(function (state) {
  if (state.visible) {
    median.android.screen.normal();       // exit fullscreen while typing
    median.statusbar.set({ overlay: false });
  } else {
    median.android.screen.fullScreen();   // restore immersion
    median.statusbar.set({ overlay: true });
  }
});
```

## Soft-prompt priming then native ask (iOS)

**Skills:** [median-push-auth](skills/median-push-auth/SKILL.md) (OneSignal IAM + register)

OneSignal in-app message as a primer; the Allow button action calls
`median.onesignal.register()`:

1. Configure auto-register OFF (plugin settings).
2. Build the soft-prompt IAM in OneSignal's HTML Composer with two buttons:
   "Allow" → runs `median.onesignal.register()`; "Maybe later" → dismiss.
3. Trigger display at a high-intent moment:
   `median.onesignal.iam.addTrigger({ promptPush: 'true' });`
4. Pause IAMs during sensitive flows:
   `median.onesignal.iam.pauseInAppMessages();` / `resumeInAppMessages()`.
5. Handle notification opens with custom data (not targetUrl) when routing
   is conditional:

```javascript
function median_onesignal_push_opened(data) {
  if (data.openUrl) myRouter.push(data.openUrl); // custom key, manual routing
}
```

## Analytics split: native SDK vs gtag

**Skills:** [median-analytics-iap](skills/median-analytics-iap/SKILL.md) (Firebase Analytics) · [median-bridge-setup](skills/median-bridge-setup/SKILL.md) (app-vs-browser UA detection)

```javascript
function logAnalyticsEvent(eventName, eventProperties) {
  if (navigator.userAgent.indexOf('median') > -1) {
    median.firebaseAnalytics.event.logEvent({ event: eventName, data: eventProperties });
  } else {
    gtag('event', eventName, eventProperties);
  }
}
```

## Social login button swap (browser vs native)

**Skills:** [median-bridge-setup](skills/median-bridge-setup/SKILL.md) (UA detection) · [median-push-auth](skills/median-push-auth/SKILL.md) (`socialLogin.*`)

```javascript
const isMedian = navigator.userAgent.indexOf("median") >= 0;
// Remove buttons whose class doesn't match the current context
function prune(cls) {
  const els = document.getElementsByClassName(cls);
  while (els.length) els[0].parentNode.removeChild(els[0]);
}
isMedian ? prune("browser-only") : prune("native-only");
```

Native buttons call `median.socialLogin.<provider>.login({ callback })`;
browser buttons keep the provider's web SDK.
