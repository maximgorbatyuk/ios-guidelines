# Analytics (Firebase / Google Analytics) Guideline for Future iOS Apps

This guideline is based on comparative analysis of analytics implementations in:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It defines a reusable standard for event tracking, user identifier handling, Debug/Release behavior, and operational setup.

---

## 1) What Current Apps Do

## 1.1 Core architecture in all apps

All apps have a singleton `AnalyticsService` with:

1. `trackEvent(name, properties)`
2. `trackScreen(screenName, properties)`
3. global properties merged into every event
4. a generated `session_id` per app run
5. a persistent `user_id` fetched from SQLite `user_settings`

All apps currently generate `user_id` via `UserSettingsRepository.fetchOrGenerateUserId()` and append it to event payloads.

## 1.2 Firebase initialization behavior

### Journey Wallet

- `FirebaseApp.configure()` is called only in non-DEBUG branch in app delegate.
- `AnalyticsService` still calls `Analytics.logEvent(...)` in Debug builds.
- Dev mode logs event payloads via `os.Logger` when `BuildEnvironment == dev`.

### EVChargingTracker

- Same pattern as Journey Wallet:
  - configure Firebase only in non-DEBUG,
  - still call `Analytics.logEvent(...)` in Debug.
- `AnalyticsService` lives in shared `BusinessLogic`, but is excluded from ShareExtension target membership.

### CreativityHub

- Strictest split:
  - `FirebaseCore` imported only for non-DEBUG app target code,
  - `FirebaseApp.configure()` only in Release branch,
  - `AnalyticsService` wraps SDK calls with `#if DEBUG` and logs locally only in Debug.

Recommendation:

- Use CreativityHub behavior as the baseline for future apps.

---

## 2) What Data Is Logged

## 2.1 Global properties (current)

### Journey Wallet and EVChargingTracker

- `session_id`
- `app_version`
- `environment`
- `platform`
- `os_version`
- `app_language`
- `user_id` (if initialized)

### CreativityHub

- `session_id`
- `app_version`
- `platform`
- `os_version`
- `user_id` (if initialized)

Gap:

- global property sets are inconsistent across apps.

## 2.2 Event categories observed

Across all apps, events mostly fall into:

1. app lifecycle (`app_opened`)
2. screen views (`trackScreen` -> GA `screen_view`)
3. UI interactions (button taps, picker changes)
4. entity CRUD actions (create/edit/delete across domain objects)
5. onboarding flow events
6. import/export/backup related actions

Most tracked properties are metadata (screen names, IDs, action names), not user content text.

---

## 3) User ID Generation and Tracking

## 3.1 Current generation flow

1. `AnalyticsService` init runs early (singleton access in app root/views).
2. It calls `UserSettingsRepository.fetchOrGenerateUserId()`.
3. If no existing ID, repository stores a new UUID string under key `user_id`.
4. `user_id` is attached to event payload global properties.

## 3.2 Current gaps

1. `identifyUser(...)` exists but is not called by default in app startup, so Firebase `setUserID` may never be set.
2. `user_id` key usage is string-literal in Journey/EV repos, while CreativityHub uses typed enum key.

Recommendation:

- Call `identifyUser(userId)` once after loading/generated ID in Release builds.

---

## 4) Cross-App Comparison

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| Firebase in Debug | SDK called, but app not configured | SDK calls skipped in DEBUG | SDK called, but app not configured | Skip SDK calls in DEBUG |
| Global props consistency | includes env + language | missing env + language | includes env + language | one shared global schema |
| user_id storage key typing | string literal | enum key (`UserSettingKey.userId`) | string literal | typed key enum |
| Service location | app target | app target | shared layer (excluded for extension) | app target preferred |
| ShareExtension isolation | naturally isolated by target | naturally isolated by target | requires pbx exception list | ensure extension-safe membership |
| Error helper | no dedicated helper | has `trackError` | no dedicated helper | include `trackError` in all apps |
| Tests | none | none | none | add analytics unit tests |

---

## 5) Gaps Found and Fixes

## Gap 1: Debug builds still invoking Firebase SDK (Journey/EV)

Risk:

- noisy logs, undefined behavior when configure is skipped, accidental telemetry if setup changes.

Fix:

- gate all SDK calls with `#if !DEBUG` (or runtime feature flag).

## Gap 2: Inconsistent global metadata schema

Fix:

- standardize global properties across apps.

## Gap 3: Static cached `app_language` may become stale

Risk:

- if user changes app language at runtime, cached global properties may continue sending old language.

Fix:

- rebuild dynamic global properties per event or invalidate cache on language change.

## Gap 4: No standard analytics tests

Fix:

- add unit tests for property merging, comparator logic, debug/release gating, and user_id initialization.

## Gap 5: CI secret handling inconsistency

Observed:

- EV `ci_post_clone.sh` prints generated plist contents (contains secrets).

Fix:

- never print secret-bearing file content in CI logs.

## Gap 6: Privacy text inconsistency with implementation

Observed:

- some policies mention anonymous analytics but not always explicit pseudonymous `user_id` usage.

Fix:

- keep privacy policy and App Store privacy answers aligned with real payload fields.

---

## 6) Canonical Analytics Architecture for New Apps

Use this structure:

1. `AnalyticsServiceProtocol`
2. `AnalyticsService` singleton (app target)
3. `AnalyticsEvent` constants (typed names)
4. `AnalyticsProperty` constants (typed keys)
5. repository-backed persistent `user_id`

Recommended API:

```swift
protocol AnalyticsServiceProtocol {
    func trackEvent(_ name: String, properties: [String: Any]?)
    func trackScreen(_ screenName: String, properties: [String: Any]?)
    func trackButtonTap(_ buttonName: String, screen: String, additionalParams: [String: Any]?)
    func trackError(_ error: Error, context: String)
}
```

Implementation requirements:

1. initialize `user_id` once and cache it,
2. set Firebase user id once (`Analytics.setUserID`) in Release,
3. merge global + event-specific params deterministically,
4. allow runtime-safe logging in Debug without network send.

---

## 7) Recommended Global Properties Schema

Send with every event:

- `session_id`
- `user_id`
- `app_version`
- `build_environment`
- `platform`
- `os_version`
- `app_language`

Optional:

- `app_bundle_id`
- `device_class` (iPhone/iPad)

---

## 8) Event Naming and Parameter Conventions

Use GA4-safe naming:

1. lowercase snake_case event names
2. stable parameter keys
3. avoid dynamic key names
4. avoid user-generated raw text fields (notes/title/body)

GA4 hard limits (enforce in code, not only by convention):

1. event name length: `1...40` characters
2. event name regex: `^[A-Za-z][A-Za-z0-9_]{0,39}$`
3. reserved event name prefixes not allowed: `firebase_`, `google_`, `ga_`
4. max custom parameters per event: `25`
5. parameter key length: `1...40`, same character rules as event names
6. string parameter values should be capped (recommended: `<=100` chars for standard GA4 properties)

Implementation guardrails:

1. add a centralized analytics validator for event names and params,
2. in Debug, assert or log validation failures,
3. in Release, drop invalid fields safely rather than crashing.

Recommended event families:

- `screen_view` (via `trackScreen`)
- `app_opened`
- `button_tapped`
- `<entity>_created|updated|deleted`
- `backup_*` / `import_*` / `export_*`
- `onboarding_*`

---

## 9) Debug vs Production Behavior (Required)

### Debug (development)

1. log analytics payloads locally via `os.Logger`.
2. do not call Firebase SDK network methods.
3. keep event structure identical to release for validation.

### Release (production)

1. call `FirebaseApp.configure()` once at app startup.
2. send events to Firebase Analytics.
3. apply `setUserID` with persistent app user id.

Lifecycle integrity rules:

1. emit `app_opened` exactly once per launch from a single root lifecycle path,
2. if manual `trackScreen` is used, set `FirebaseAutomaticScreenReportingEnabled = NO` in app `Info.plist` to avoid duplicate `screen_view` events.

---

## 10) Firebase / GA Setup Steps (Web Consoles)

## 10.1 Firebase Console

1. Create Firebase project.
2. Enable Google Analytics for project.
3. Add iOS app registrations for bundle IDs:
   - release bundle ID,
   - debug/dev bundle ID (recommended separate registration for test telemetry).
4. Download `GoogleService-Info.plist` for each iOS app registration.

## 10.2 GA4 (inside Firebase)

1. Verify GA4 property + data stream are active.
2. Register custom dimensions for important event params (for example `session_id`, `user_id` if needed for analysis policy).
3. Define conversion events only for meaningful product signals.

## 10.3 Local development setup

1. Store secrets in `scripts/.env` (gitignored).
2. Generate `GoogleService-Info.plist` from script.
3. Never commit plist or `.env`.

## 10.4 Xcode Cloud setup

1. Add secrets in workflow environment:
   - `FIREBASE_API_KEY`
   - `FIREBASE_GCM_SENDER_ID`
   - `FIREBASE_APP_ID`
2. Generate `GoogleService-Info.plist` in `ci_post_clone.sh`.
3. Do not print plist contents in CI logs.

## 10.5 Script and secret hardening (required)

1. CI scripts must fail fast when required secrets are missing.
2. Never output secret-bearing file contents (`cat GoogleService-Info.plist`) in CI logs.
3. Ensure ignore rules include:
   - `scripts/.env`
   - `ci_scripts/.env`
   - `GoogleService-Info.plist`
   - `**/GoogleService-Info.plist`
4. Keep plist generation bundle-ID-aware for Debug/Release targets (do not rely on stale hardcoded IDs).
5. Add a CI check that fails if `GoogleService-Info.plist` or `.env` files are tracked.

---

## 11) ShareExtension Considerations

Analytics for app and extension should be intentionally designed, not accidental.

Rules:

1. If extension analytics is not required, do not include AnalyticsService in extension target.
2. If `BusinessLogic` is shared, explicitly exclude extension-unavailable analytics files from extension membership.
3. Keep extension and app event taxonomies distinct if extension telemetry is later added.
4. Never let analytics initialization block extension startup.

---

## 12) Privacy and Compliance Checklist

Before release:

1. Privacy policy explicitly mentions analytics provider and data classes collected.
2. If `user_id` is sent, describe it as pseudonymous app-generated identifier.
3. App Store Connect privacy questionnaire matches real telemetry.
4. No user content text (notes/documents/descriptions) is sent in analytics payloads.
5. Do not log persistent identifiers (`user_id`) in Release logs.
6. Do not send raw `error.localizedDescription` without sanitization; prefer stable error codes/categories.
7. Define consent/opt-out behavior explicitly (where required by policy or region) and document runtime collection toggle behavior.

---

## 13) Test Matrix

### Unit tests

1. global property merge precedence
2. user_id initialization path (existing vs new)
3. Debug gating (no Firebase SDK calls)
4. Release gating (Firebase calls allowed)
5. language/environment property correctness
6. event name validator (`<=40`, allowed chars, reserved prefix rejection)
7. parameter validator (count/key length/value length boundaries)
8. reserved global keys (`session_id`, `user_id`, etc.) cannot be overridden by per-event params
9. `app_opened` dedup (one event per launch)
10. release path calls `identifyUser(...)` exactly once after user ID initialization

### Integration tests

1. app startup config path (release only)
2. screen tracking helper emits proper GA screen params
3. button tracking helper emits expected keys
4. manual screen tracking + `FirebaseAutomaticScreenReportingEnabled = NO` produces no duplicate screen events
5. release smoke test verifies incoming events in GA DebugView (`-FIRDebugEnabled`)

### Regression checks

1. no crashes when Firebase plist missing in Debug
2. no telemetry in Debug builds
3. telemetry present in Release with valid plist
4. CI logs contain no secret-bearing plist output
5. repository contains no tracked `.env` or `GoogleService-Info.plist` files
6. no analytics event names over 40 chars in source scan

---

## 14) Definition of Done

Analytics implementation is complete only if:

1. Debug and Release behavior are intentionally separated,
2. persistent `user_id` is generated, stored, and attached correctly,
3. global metadata schema is stable and documented,
4. event taxonomy is consistent and localized to product semantics,
5. CI/local secret handling is secure,
6. privacy documentation matches actual payloads,
7. ShareExtension target boundaries are explicitly enforced,
8. GA4 naming/parameter limits are automatically validated,
9. manual-vs-automatic screen tracking cannot produce duplicate `screen_view`,
10. root lifecycle prevents duplicate `app_opened`,
11. persistent identifiers are not written to release logs,
12. event-level payloads cannot override protected global keys.

---

## 15) Recommended Defaults for New Apps

Use these defaults unless product requirements differ:

- `AnalyticsService` in app target only
- `#if DEBUG` local logging, no Firebase network calls
- release-only `FirebaseApp.configure()`
- persistent SQLite-backed `user_id` + `session_id`
- standardized global properties schema
- typed event/property constants
- centralized event/param validator with GA4 limits
- protected global keys (no event-level override for `session_id`/`user_id`)
- `FirebaseAutomaticScreenReportingEnabled = NO` when using manual `trackScreen`
- one-time `app_opened` emission per launch
- no release logging of stable identifiers
- unit tests for service behavior and initialization

---

## 16) Second-Pass Gaps Found (Mar 2026)

Observed in current workspace analysis:

1. One event name exceeds GA4 40-char limit:
   - `expenses_chart_filter_maintenance_selected` in
     `EVChargingTracker/EVChargingTracker/ChargingSessions/ExpensesChartViewModel.swift`
2. `identifyUser(...)` helper exists in all apps but is not called from startup flow by default.
3. Manual `trackScreen(...)` is used widely, while `FirebaseAutomaticScreenReportingEnabled` key is not set in current plist files (risk of duplicate screen events).
4. `EVChargingTracker/ci_scripts/ci_post_clone.sh` prints generated plist contents in logs (secret exposure risk).
5. `EVChargingTracker/ci_scripts/ci_post_clone.sh` does not currently fail fast on missing Firebase secrets.
6. `EVChargingTracker/.gitignore` currently lacks `scripts/.env` and `ci_scripts/.env` ignore rules.
7. `user_id` logging appears in initialization logs in current analytics services (should be blocked in Release).
8. current merge behavior allows event-level properties to override global keys if same key is passed.

These findings are the reason sections 8-15 now include explicit validation, lifecycle, and secret-handling rules.
