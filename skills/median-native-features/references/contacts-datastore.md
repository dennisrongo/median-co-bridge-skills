# Native Contacts + Native Datastore (full reference)

## median.contacts.* (Native Contacts) — plugin required

Source: https://docs.median.co/docs/native-contacts

**Purpose**: Sync/pick device contacts for form completion, CRM entry, lookups by email/phone.

**Prerequisite**: Native Contacts plugin enabled. App auto-prompts for permission on first access.

### Call signatures (each accepts a callback or returns a Promise)

```javascript
median.contacts.getPermissionStatus({ callback: myCallback })
await median.contacts.getPermissionStatus({})            // → { "status": "STRING" }

median.contacts.getAll({ callback: myCallback })
await median.contacts.getAll({})

median.contacts.pickContact({ callback: myCallback, multiple: BOOL })   // native picker UI
await median.contacts.pickContact({ multiple: false })
```

### Parameters

| Param | Type | Description |
|---|---|---|
| `callback` | function | For callback style |
| `multiple` | boolean | `true` = multi-select, `false` = single contact only (pickContact) |

### Permission status values

| Platform | Values |
|---|---|
| iOS | `granted`, `denied`, `restricted` (admin prohibited, e.g. parental controls/MDM), `notDetermined` |
| Android | `granted`, `denied` (not yet asked OR explicitly denied) |

### Response shape

`{ success: true, contacts: [...] }` when permission granted.

### Contact fields

- **iOS**: `birthday`, `namePrefix`, `givenName`, `middleName`, `familyName`, `previousFamilyName`, `nameSuffix`, `nickname`, `phoneticGivenName`, `phoneticMiddleName`, `phoneticFamilyName`, `organizationName`, `departmentName`, `jobTitle`, `note` (pickContact only), `phoneNumbers[]`, `emailAddresses[]`, `postalAddresses[]`
- **Android**: `birthday`, `givenName`, `familyName`, `companyName`, `companyTitle`, `note`, `phoneNumbers[]`, `emailAddresses[]`, `postalAddresses[]`

### Array field schemas (verbatim)

```javascript
// phoneNumbers
{ "label": "STRING", "phoneNumber": "STRING" }

// emailAddresses
{ "label": "STRING", "emailAddress": "STRING" }

// postalAddresses — all strings
{ "label", "street", "city", "state" /* iOS only */, "region" /* Android only */, "postalCode", "country", "isoCountryCode" /* iOS only */, "subAdministrativeArea" /* iOS only */, "subLocality" /* iOS only */ }
```

### Gotchas

- iOS `note` field not returned by `getAll()` by default — needs the `com.apple.developer.contacts.notes` entitlement (Apple approval).
- `note` IS available via `pickContact()` on iOS without extra entitlement.
- `pickContact` is the recommended approach for explicit user selection.

---

## median.storage.app / median.storage.cloud (Native Datastore) — plugin required

Source: https://docs.median.co/docs/native-datastore

**Purpose**: Persist key-value data on-device (App Storage) or synced to the user's cloud account (Cloud Storage).

**Backends**: App Storage = Android SharedPreferences / iOS UserDefaults. Cloud Storage = Android SharedPreferences + Android Backup Service / iOS Apple Keychain Services.

### Call signatures (identical shape for `app` and `cloud`)

```javascript
median.storage.app.set({ key: KEY, value: VALUE, statuscallback: statcb })
median.storage.app.get({ key: KEY, callback: cbRead })       // callback style
median.storage.app.get({ key: KEY }).then(...)               // promise style
median.storage.app.delete({ key: KEY, statuscallback: statcb })
median.storage.app.deleteAll({ statuscallback: statcb })
// same four under median.storage.cloud.*
```

### Parameters

| Call | Param | Type | Required | Description |
|---|---|---|---|---|
| set | `key` | String | Required | The key to store the value under |
| set | `value` | String | Required | The value to persist |
| set | `statuscallback` | Function | No | Receives `{ status }` |
| get | `key` | String | Required | The key to retrieve |
| get | `callback` | Function | Required for callback style | Receives `result.data` and `result.status` |

### Response shape

- `get` callback/promise → `{ data: <stored value>, status: STRING }`
- `set`/`delete`/`deleteAll` statuscallback → `{ status: STRING }`

### Status values

`success`, `read-error`, `write-error`, `delete-error`, `preference-not-found`

### Storage limits

| Platform | App Storage | Cloud Storage |
|---|---|---|
| Android | No limit | Up to 5 MB |
| iOS | Up to 500 KB | Up to 16 MB |

### Platform notes / Gotchas

- Android cloud sync via BackupManager is asynchronous and system-scheduled — no guaranteed sync time.
- iOS Keychain sync depends on the user's iCloud Keychain settings.
- Keys are case-sensitive.
- Android file locations: `DATA/data/APP_PACKAGE_NAME/shared_prefs/user_preferences.xml` (app) and `user_preferences_backup.xml` (cloud).
- Cloud storage survives reinstall (iOS Keychain is hardware-backed; suitable for sensitive values).

### Minimal snippet

```javascript
median.storage.app.set({ key: 'theme', value: 'dark', statuscallback: statcb });
median.storage.app.get({ key: 'theme' }).then(function(result) {
  console.log(result.data); console.log(result.status);
});
median.storage.cloud.deleteAll({ statuscallback: statcb });
```
