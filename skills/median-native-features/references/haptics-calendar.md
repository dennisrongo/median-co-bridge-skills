# Haptics + Calendar (full reference)

## median.haptics.trigger + median_device_shake (Haptics) — plugin required

Source: https://docs.median.co/docs/haptics

**Purpose**: Trigger haptic vibration effects (impact/notification feedback) and respond to device shake gesture.

**Prerequisite**: Haptics plugin enabled under Native Plugins (App Studio). Physical device required — no meaningful feedback in simulators.

### Call signatures

```javascript
median.haptics.trigger({ style: styleName })
function median_device_shake() {...}   // page-defined; invoked when user shakes the device
```

The shake function is also insertable via Custom JavaScript in App Studio.

### Parameters

`style` string:

| Platform | Styles |
|---|---|
| iOS + Android | `impactLight`, `impactMedium`, `impactHeavy`, `notificationSuccess`, `notificationWarning`, `notificationError` |
| Android only | `tick`, `click`, `double_click` |

### Callback/promise

None documented for trigger.

### Gotchas

- Android-only styles have no effect on iOS.
- Function name must match exactly: `median_device_shake`.
- NPM listener alternative: `Median.deviceShake.addListener(() => {...})`.

### Minimal snippet

```javascript
median.haptics.trigger({ style: 'impactMedium' });
function median_device_shake(){
  document.querySelector('.sideNavigation').style.visibility = "visible";
};
```

---

## Calendar (plugin required — .ics link interception, no bridge call)

Source: https://docs.median.co/docs/calendar

**Purpose**: Let users add events to their device calendar by tapping `.ics` or `data:text/calendar` links.

**Prerequisite**: Calendar plugin enabled (plug-and-play) + JS Bridge enabled.

### Call signature

**None — the plugin intercepts link taps.** Flow: (1) intercepts navigation, (2) downloads the `.ics`, (3) parses it, (4) prompts user to confirm adding to native calendar.

### Two implementation paths

1. Hosted `.ics` file:

```html
<a href="https://yoursite.com/event.ics">Add to Calendar</a>
```

2. Data URI (verbatim docs example):

```html
<a href="data:text/calendar;charset=utf8,BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
DTSTART:20240117T190000Z
DTEND:20240117T200000Z
SUMMARY:Doctor Appointment
DESCRIPTION:Annual checkup
LOCATION:123 Main St
END:VEVENT
END:VCALENDAR">Add to Calendar</a>
```

### Gotchas

- Malformed iCalendar data → missing/incorrect event details; validate required fields `DTSTART`, `DTEND`, `SUMMARY`.
- Not a standalone web calendar — only intercepts `.ics` links.
- There is no `median.calendar.*` API — do not invent bridge calls for calendar.
