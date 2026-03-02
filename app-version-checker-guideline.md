# App Version Checker Guideline for Future iOS Apps

This guideline is based on analysis of current implementations in:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It defines a reusable standard for checking App Store updates, showing in-app update prompts/badges, and handling ShareExtension-related target considerations.

---

## 1) What Current Apps Do

### 1.1 Shared core approach

All three apps use the same flow:

1. Read installed app version from `EnvironmentService.getAppVisibleVersion()`.
2. Extract short version token from `<version> (<build>)`.
3. Call iTunes Lookup API:
   - `https://itunes.apple.com/lookup?id=<APP_STORE_ID>`
4. Parse `results[0].version`.
5. Compare local version and store version.
6. If update is available:
   - show update row in Settings,
   - optionally show Settings tab badge.

### 1.2 Where checks are triggered

- All apps trigger check from `MainTabView` `.onAppear` via `MainTabViewModel.checkAppVersion()`.
- Result is passed to settings UI as `showAppUpdateButton` flag.

### 1.3 How update is shown

- `journey-wallet`: highlighted settings row + Settings tab badge (`"New!"`).
- `EVChargingTracker`: highlighted settings row + Settings tab badge (`"New!"`).
- `CreativityHub`: highlighted settings row; no tab badge currently wired in `MainTabView`.

### 1.4 Current comparison logic

All three effectively use inequality (`currentVersion != appStoreVersion`).

This is simple but not semantically correct for all cases.

---

## 2) Cross-App Comparison

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| Checker location | App target (`JourneyWallet/Services`) | App target (`CreativityHub/Services`) | Shared layer (`BusinessLogic/Services`) | Keep checker app-target only |
| Checker type | class + protocol | final class + protocol | class + protocol | protocol + final class/actor |
| Version compare | `!=` | `!=` | `!=` | semantic `store > local` only |
| Query anti-cache params | yes (`current`, `now`) | yes (`current`, `now`) | yes (`current`, `now`) | keep or replace with TTL cache |
| Tab badge | yes | no | yes | add optional badge policy |
| Settings update row | yes | yes | yes | keep highlighted row |
| Localization style | key-like raw phrase (`"App update available"`) | namespaced key (`settings.update_available`) | key-like raw phrase (`"App update available"`) | use namespaced keys everywhere |
| App Store link opening | ViewModel + `UIApplication.shared.open` | View + `openURL` + env link | ViewModel + `UIApplication.shared.open` | prefer `openURL` from View layer |
| Tests | none | none | none | add checker tests + comparator tests |

---

## 3) Gaps Found

### Gap 1: Incorrect update decision rule

Current rule: `local != store`.

Problem:

- if local build is newer than currently published store version (common in debug/TestFlight), app still says "update available".

Required fix:

- show update only when `storeVersion > localVersion`.

### Gap 2: No semantic version comparator

No dedicated comparator for:

- dotted versions (`1.2.10` vs `1.2.9`),
- date-style versions (`2026.1.8`),
- uneven component lengths.

Required fix:

- implement deterministic normalized numeric compare.

### Gap 3: Missing throttling/persistence

Checks run every `MainTabView` appearance.

Required fix:

- cache check result + timestamp (for example TTL 6-24h) in lightweight local storage.

### Gap 4: Inconsistent localization key quality

- Journey/EV use non-namespaced key text in code (`L("App update available")`).
- CreativityHub uses clean namespaced key.

Required fix:

- standardize to `settings.update_available` and related keys.

### Gap 5: No automated tests

No unit tests for checker parsing/compare behavior or network error handling.

Required fix:

- add protocol-injected URL session/mock transport and comprehensive tests.

### Gap 6: ShareExtension target coupling risk

EV checker sits in `BusinessLogic` (shared with extension target).

Risk:

- app-only feature compiled into extension unnecessarily.

Required fix:

- keep checker in app target module or explicitly exclude from ShareExtension membership.

---

## 4) Canonical Architecture for New Apps

Use this structure:

1. `AppVersionCheckerProtocol`
2. `AppVersionChecker` service (app target)
3. `VersionComparator` helper (pure function, testable)
4. `MainTabViewModel` orchestration method
5. `MainTabView` state (`showAppVersionBadge`)
6. `UserSettingsView(showAppUpdateButton:)`

### Recommended method contract

```swift
protocol AppVersionCheckerProtocol {
    func checkAppStoreVersion() async -> AppUpdateCheckResult
}

enum AppUpdateCheckResult {
    case updateAvailable(storeVersion: String)
    case upToDate
    case unavailable   // missing app id, no store data, network fail
}
```

Use explicit result type instead of `Bool?` for clarity and observability.

---

## 5) Version Comparison Standard

### Rule

Only show update when App Store version is newer:

- `storeVersion > localVersion` => update available
- `storeVersion == localVersion` => no update
- `storeVersion < localVersion` => no update

### Comparator requirements

1. Split by `.`
2. Parse each component as integer (non-numeric -> 0 fallback or strict reject)
3. Normalize lengths with trailing zeros
4. Compare component by component

Works for:

- `1.2` vs `1.2.0`
- `1.2.10` vs `1.2.9`
- `2026.1.8` vs `2026.1.10`

---

## 6) API and Networking Standard

### Endpoint

- Use iTunes Lookup API:
  - `https://itunes.apple.com/lookup?id=<APP_STORE_ID>`

Optional query params for cache busting are acceptable, but app-level TTL caching is still recommended.

### Required checks

1. Validate `APP_STORE_ID` exists and non-empty.
2. Validate HTTP status code is `200`.
3. Parse JSON safely (`resultCount > 0`, `results.first.version`).
4. Handle "no results" as `unavailable` (not crash).

### Logging

- Log concise diagnostics in debug/dev.
- Avoid dumping full payload in production logs.

---

## 7) UI Behavior Standard

### Settings page

When update is available:

1. Show highlighted update row near top of settings.
2. Include action button that opens App Store page.
3. Track click analytics event (optional but recommended).

### Tab badge

Recommended:

- Show settings tab badge when update is available.
- Use concise badge text (`"New!"`) or dot badge.

### Badge dismissal policy

Define explicit policy (required):

1. clear when user taps update button, or
2. clear when checker returns up-to-date, or
3. clear manually in session after opening Settings.

---

## 8) Localization Keys (Required)

Use namespaced keys for all update UI:

- `settings.update_available`
- `settings.update_action`
- `settings.update_badge` (if text badge localized)
- `settings.update_unavailable` (optional fallback info)

Do not use free-text pseudo keys like `"App update available"` in new code.

---

## 9) Caching and Frequency Policy

Recommended default:

1. Check once on app startup/session start.
2. Cache result for 12 hours.
3. Allow manual re-check in developer mode.

Store minimal cache:

- `lastCheckAt`
- `lastStoreVersion`
- `lastResult`

---

## 10) ShareExtension Considerations

Even though this is an app feature, include these rules in apps with ShareExtension:

1. Keep `AppVersionChecker` app-target only where possible.
2. Do not call version check from extension flow.
3. If checker lives in shared folder, exclude it from extension target membership unless intentionally needed.
4. Keep `APP_STORE_ID` in main app metadata; extension does not require update-check UI.
5. Ensure app version check does not block or affect ShareExtension migration flow.

---

## 11) Apple Website / Console Steps

These are required for update-check feature to function correctly.

### 11.1 App Store Connect

1. Create app record in App Store Connect.
2. Obtain numeric Apple ID (App Store ID) for the app.
3. Ensure app has at least one public App Store release (lookup may return no result before public availability).
4. Keep versioning consistent (`MARKETING_VERSION`, `CURRENT_PROJECT_VERSION`).
5. Verify app is available in storefront(s) you test against.

### 11.2 Project wiring

1. Put `APP_STORE_ID` in `Config/Base.xcconfig`.
2. Expose it via Info.plist key `AppStoreId`.
3. Read via `EnvironmentService.getAppStoreId()`.
4. Build App Store link via `https://apps.apple.com/app/id<APP_STORE_ID>`.

### 11.3 Apple Developer Portal

No special capability/entitlement is required for version checking itself.

For apps with ShareExtension, continue normal App Group/App ID/provisioning setup (independent from this feature).

---

## 12) Test Matrix

### Unit tests

1. comparator: equal versions
2. comparator: store newer
3. comparator: local newer
4. comparator: uneven component counts
5. checker: missing `APP_STORE_ID`
6. checker: network error
7. checker: invalid/missing JSON fields
8. checker: successful parse + update available

### Integration tests

1. `MainTabView` sets `showAppVersionBadge` on startup.
2. `UserSettingsView` shows/hides update row correctly.
3. Update button opens valid App Store URL.
4. Badge policy behaves as specified.

### Regression checks

1. Debug build should not crash if App Store record unavailable.
2. Release build should not show false-positive update when local > store.
3. Localization keys exist for all supported languages.

---

## 13) Definition of Done

Feature is done only if:

1. App Store check runs reliably with graceful fallback,
2. update decision is semantic (`store > local`),
3. settings update UI is localized and functional,
4. optional badge behavior is implemented consistently,
5. tests cover comparator and checker edge cases,
6. ShareExtension target remains unaffected by app-only update logic.

---

## 14) Recommended Defaults for New Apps

Use these defaults unless product requirements differ:

- check on app startup via `MainTabViewModel`
- TTL cache 12h
- semantic version comparator
- settings row + settings tab badge
- namespaced localization keys
- app-only checker target membership
- App Store ID from xcconfig/Info.plist (never hardcoded)
