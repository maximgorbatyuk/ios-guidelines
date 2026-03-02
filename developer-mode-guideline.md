# Developer Mode Guideline for Future iOS Apps

This document is based on analysis of Developer Mode implementations in:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It defines a reusable standard for building Developer Mode in future apps, including tools that are especially useful for ShareExtension development and QA.

---

## 1) Purpose

Developer Mode should provide hidden, safe, test-focused tools that:

1. help QA and developers validate data flows quickly,
2. speed up ShareExtension verification (App Group migration, shared storage),
3. allow controlled destructive/reset operations in non-production contexts.

Developer Mode is not a user feature and must remain hidden by default.

---

## 2) Cross-App Comparison Summary

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| Activation | 15 taps on app version row | 15 taps on app version row | 15 taps on app version row | Keep exactly 15 taps |
| Manager style | `ObservableObject` + `@Published` singleton | `@Observable` + `@MainActor` singleton | `ObservableObject` + `@Published` singleton | Prefer `@MainActor` + testable singleton manager |
| Persistence | In-memory only | In-memory only | In-memory only | Keep in-memory (auto-off on app relaunch) |
| Developer section visibility | only when special developer mode enabled | only when special developer mode enabled | only when special developer mode enabled | Keep hidden unless explicitly unlocked |
| Localization quality | Mixed (some hardcoded dev strings) | Strong (mostly localized) | Mixed (many hardcoded dev strings) | Localize all user-facing dev text |
| User settings table viewer | Inline view in settings file | Dedicated `UserSettingsTableView` | Dedicated `UserSettingsTableView` | Use dedicated reusable view |
| Document storage browser | Yes (sync loading) | Yes (async/background loading + better errors) | Yes (sync loading) | Use CreativityHub async approach |
| Random data tool | Journey-scoped generator service | Project-scoped generator service | Car-scoped generation in ViewModel | Use dedicated generator service (not bloated ViewModel) |
| Migration debug tool | Reset App Group migration flag | Reset migration records + schema version row | Migration status + reset App Group migration flag | Include status + safe reset + clear warnings |
| Tests for manager | None | Yes (`DeveloperModeManagerTests`) | None | Add manager tests in every app |

### Current Feature Inventory (Observed)

| Developer Feature | Journey Wallet | CreativityHub | EVChargingTracker |
|---|---|---|---|
| Hidden 15-tap activation on version row | Yes | Yes | Yes |
| Disable developer mode row | Yes | Yes | Yes |
| Activation success alert | Yes | Yes | Yes |
| Request notification permission | Yes | Yes | Yes |
| Send immediate test notification | Yes | Yes | Yes |
| Schedule delayed test notification | Yes | Yes | Yes |
| View `user_settings` table | Yes | Yes | Yes |
| Document storage browser | Yes | Yes | Yes |
| Launch screen preview action | Yes (raw launch view) | Yes (dedicated preview wrapper) | Yes (raw launch view) |
| Delete all data action | Yes | Yes | Yes |
| Scoped delete action | Journey-scoped deletion inside random generator | Project-scoped deletion inside random generator | Explicit "Delete car expenses" |
| Random data generation | Yes (journey picker) | Yes (project picker) | Yes (selected car, inline generator logic) |
| Migration status row | No | Schema version only | Yes (migrated/legacy/app group) |
| Reset App Group migration flag | Yes | No | Yes |
| Reset schema migration records | No | Yes | No |

---

## 3) Activation Contract (Must Follow)

Use this exact behavior:

1. Developer Mode is disabled by default.
2. In `Settings -> About app`, tapping app version row increments hidden tap counter.
3. Threshold is `requiredTaps = 15`.
4. On threshold:
   - set `isDeveloperModeEnabled = true`,
   - set `shouldShowActivationAlert = true`,
   - reset tap counter to `0`.
5. If already enabled, repeated taps must not re-trigger activation alert.
6. Add explicit disable action in settings (Developer Mode row).
7. State is in-memory only (disabled again after app restart).

Recommended manager API:

- `handleVersionTap()`
- `enableDeveloperMode()`
- `disableDeveloperMode()`
- `dismissAlert()`

---

## 4) Access Rules and Safety Gates

Use two checks:

1. `isDeveloperModeEnabled` (special unlocked mode via 15 taps) for UI visibility.
2. `isDevelopmentMode` (build env dev OR special mode) for operation guards.

Rules:

- Developer section should be shown only when `isDeveloperModeEnabled == true`.
- Every destructive/test function must also guard internally:
  - if `!isDevelopmentMode`, abort and log.
- Do not rely only on UI visibility for safety.

---

## 5) Required Feature Set

Future apps should implement the full feature set below (or explicitly justify exclusions).

### 5.1 Notification Test Tools

Actions:

1. Request notification permission
2. Send immediate test notification
3. Schedule delayed test notification (for example 5 seconds)

Why:

- validates notification entitlement flow and local notification wiring quickly.

### 5.2 User Settings Table Viewer

Show raw `user_settings` records:

- columns: id, key, value
- read-only
- supports text selection/copy for debugging
- empty state if table is empty

Implementation notes:

- Use dedicated screen/sheet (`UserSettingsTableView`), not inline giant section.
- Fetch via `UserSettingsRepository` (`fetchAll` / equivalent DTO API).

### 5.3 Document Storage Browser (Must-Have)

Provide file-system inspector for app documents directory:

1. storage stats (total size, files count, folders count)
2. folders list + root files list
3. folder drill-down
4. file details (type, size, created, modified, path)
5. Quick Look preview
6. share file action
7. delete file action with confirmation

Implementation notes:

- Root should point to app document storage service path in App Group when ShareExtension is used.
- Use background loading to avoid UI stalls (CreativityHub pattern).
- On delete failure, show explicit error state and reload list.

### 5.4 Random Data Generator

Support entity-scoped random data generation:

- Journey Wallet: per journey
- CreativityHub: per project
- EVChargingTracker: per car

Required behavior:

1. user picks target entity in a modal picker,
2. show destructive confirmation,
3. generator deletes existing scoped data first,
4. inserts realistic randomized data for major entities,
5. logs counts and completion.

Architecture recommendation:

- Use dedicated `RandomDataGenerator` service class.
- Keep heavy generation logic out of Settings ViewModel.

### 5.5 Destructive Data Actions

Required actions (as applicable):

1. Delete all data
2. Delete scoped data (for example selected car expenses)

Rules:

- Always confirmation dialog with destructive role.
- Guard by `isDevelopmentMode`.
- Refresh UI after deletion.
- Never expose these actions outside Developer Mode UI.

### 5.6 Launch Screen QA Preview

Developer action to preview launch screen in sheet.

Preferred implementation:

- wrapper preview view with explicit close button (CreativityHub approach),
- avoid needing app relaunch to validate launch visuals.

### 5.7 Migration Debug Tools

For apps using App Group migration (especially with ShareExtension), include:

1. migration status row:
   - migration complete flag
   - legacy DB existence
   - App Group config status
2. reset migration flag action (QA only) with strong warning confirmation

Optional:

- reset migration records table (schema migration debug), but clearly separate from App Group file migration reset.

### 5.8 Schema Diagnostics

Show current DB schema version in developer section.

Why:

- helps verify expected migration set on real devices quickly.

---

## 6) ShareExtension-Focused Developer Tools

To support ShareExtension implementation and QA, Developer Mode should include:

1. App Group migration status panel (required)
2. Reset App Group migration marker action (required)
3. Document storage browser in shared container (required)
4. user_settings table viewer (required)
5. Optional ShareExtension diagnostics view:
   - extension bundle ID,
   - app group identifier,
   - shared DB path,
   - shared docs path,
   - parser last input type (if instrumented in debug)

This shortens verification cycles for extension-first launch and migration edge cases.

---

## 7) Localization Standard for Developer Mode

Observed issue:

- Journey Wallet and EVChargingTracker still contain hardcoded English strings in Developer Mode UI.

Required for future apps:

1. Localize all user-facing developer text (`L("settings.developer.*")`).
2. Keep dedicated key namespace:
   - `settings.developer.*`
   - `developer.document_storage.*`
3. Localize alerts, button labels, confirmation dialogs, and empty states.
4. Keep message clarity for destructive actions and migration resets.

---

## 8) Architecture and File Organization

Recommended structure:

```text
AppTarget/
  Services/
    DeveloperModeManager.swift
    RandomDataGenerator.swift
  Developer/
    UserSettingsTableView.swift
    DocumentStorageService.swift
    DocumentStorageBrowserView.swift
    FolderContentsView.swift
    FileDetailView.swift
    LaunchScreenPreviewView.swift
  UserSettings/
    UserSettingsView.swift
    UserSettingsViewModel.swift
```

Rules:

- Keep Developer tools app-target only.
- Do not compile developer-only UI into ShareExtension target.
- Keep `UserSettingsViewModel` orchestration-focused; move heavy logic to services.

---

## 9) Frequent Issues and Fix Patterns

### Issue 1: Hardcoded developer strings

- Seen in Journey Wallet and EVChargingTracker developer sections.
- Fix: enforce localization key usage in developer UI too.

### Issue 2: Heavy file browsing on main thread

- Journey Wallet and EVChargingTracker browser load paths are synchronous.
- Fix: use background loading and async state updates (CreativityHub pattern).

### Issue 3: Incomplete operation guards

- Some debug actions can miss explicit `isDevelopmentMode` guard if called directly.
- Fix: guard inside each ViewModel/service function, not only UI.

### Issue 4: Bloated Settings ViewModel

- EV random generation logic lives directly in `UserSettingsViewModel`.
- Fix: extract to dedicated generator service.

### Issue 5: Missing unit tests for manager behavior

- Only CreativityHub has `DeveloperModeManager` tests.
- Fix: add tests for all apps:
  - activation before/after 15 taps,
  - no reactivation while enabled,
  - disable resets state,
  - dismiss alert behavior.

### Issue 6: Migration reset confusion

- Schema migration reset and App Group migration reset are different operations.
- Fix: separate labels and warnings clearly in UI.

---

## 10) Apple Developer Website Steps (ShareExtension-Relevant)

Developer Mode itself does not require Apple portal setup, but ShareExtension QA tools do.

When app includes ShareExtension, complete these steps:

### A. Create App Groups

Apple Developer -> Certificates, Identifiers & Profiles -> Identifiers -> App Groups

Create:

- production group: `group.<team>.<app>`
- development group: `group.<team>.<app>.dev`

### B. Create/verify App IDs

Create explicit App IDs for:

1. main app release
2. main app debug (if separate)
3. share extension release
4. share extension debug (if separate)

### C. Enable App Groups capability on all relevant IDs

Recommended strict mapping:

- app release -> prod group
- app debug -> dev group
- extension release -> prod group
- extension debug -> dev group

### D. Regenerate provisioning profiles

After capability edits:

- regenerate development and distribution profiles for app and extension,
- refresh signing in Xcode.

### E. Final portal validation before release

1. group IDs in portal exactly match entitlements,
2. extension App IDs are explicit (no wildcard),
3. archive contains embedded `.appex` and valid signing for extension target.

---

## 11) App Store Connect Steps (ShareExtension + Developer QA Context)

1. Mention ShareExtension capability in release notes.
2. Update App Privacy answers if extension changes data handling.
3. Add App Review notes with clear share-sheet validation steps.
4. Validate in TestFlight on:
   - clean install,
   - upgrade install,
   - extension-first launch after upgrade.

---

## 12) Test Matrix for Developer Mode

### Manager behavior

1. 14 taps -> not enabled
2. 15 taps -> enabled + alert shown + tap count reset
3. enabled + 15 taps -> no reactivation alert
4. disable -> mode off + tap count reset

### Settings integration

1. version row tap triggers manager
2. developer section hidden by default
3. section appears only when special mode enabled

### Destructive actions

1. blocked when not in development mode
2. confirmation required
3. UI refreshes after completion

### Random data

1. entity picker works
2. generation replaces target scope only
3. generated entities are queryable in app UI

### Document browser

1. stats/folder/file listing works
2. quick look works for supported files
3. delete/share actions work
4. errors are surfaced (no silent fail)

### ShareExtension diagnostics

1. migration status reflects real state
2. migration reset re-runs on next launch
3. shared storage files visible in browser

---

## 13) Definition of Done

Developer Mode implementation is complete only if:

1. hidden 15-tap activation works,
2. all destructive actions are guarded and confirmed,
3. user settings viewer, document browser, random data, and launch screen preview are implemented,
4. migration diagnostics/reset tools are present for ShareExtension apps,
5. developer UI is fully localized,
6. manager behavior has automated tests,
7. release build keeps developer tools inaccessible until explicit unlock.

---

## 14) Suggested Defaults for New Apps

If no special constraints are provided, use this baseline:

- 15-tap in-memory Developer Mode activation
- Developer section shown only after unlock
- Notification test actions + user settings table viewer
- Async document storage browser + file detail tools
- Random data generator service scoped by selected entity
- Delete-all + scoped-delete actions with strong confirmations
- App Group migration status + reset action for ShareExtension apps
- Full localization for developer strings
- Unit tests for `DeveloperModeManager`

---

## 15) Gaps Found in Second Analysis Pass

The following gaps were identified after a deeper re-check of all three codebases.

1. **Build mode vs unlocked mode is conflated in some apps**
   - Example pattern: `isDevelopmentMode = env.isDevelopmentMode || developerModeEnabled` is reused for UI labels like "Build: Development".
   - Risk: release build can display misleading build state after developer unlock.

2. **Some actions rely only on UI gating, not function-level guards**
   - Example pattern found: random data action path without explicit `isDevelopmentMode` guard in function body.
   - Risk: accidental invocation from future code paths.

3. **Developer file browser may perform synchronous file I/O on main thread**
   - Present in Journey Wallet and EVChargingTracker variants.
   - Risk: jank/hangs on large storage trees.

4. **Developer actions still contain hardcoded strings in some apps**
   - Seen in EVChargingTracker and parts of Journey Wallet.
   - Risk: inconsistent UX and untranslated alerts.

5. **Developer file deletion can orphan DB metadata**
   - Browser deletes raw files directly; metadata rows may remain.
   - Risk: later preview/open failures that look like data corruption.

6. **Developer action analytics may pollute production metrics**
   - Some apps track developer-tapped events through common analytics paths.
   - Risk: distorted product analytics if unlock happens in production.

7. **Test coverage is inconsistent outside CreativityHub**
   - Manager behavior tests are missing in Journey Wallet and EVChargingTracker.

---

## 16) Mandatory Rules to Close These Gaps

### 16.1 Separate environment flags

Keep these flags distinct in ViewModel/service design:

- `isDebugBuild` (from environment/build config)
- `isDeveloperModeEnabled` (15-tap unlock state)
- `isDevelopmentMode` (operational gate, if needed)

Do not derive user-visible build labels from unlock state.

### 16.2 Dual gating for dangerous actions

For destructive/mutating tools (delete all data, reset migrations, generate random data):

1. Feature visible only in Developer Mode section.
2. Function-level guard (`guard isDevelopmentMode else { ... }`).
3. Confirmation dialog required.
4. Prefer a second safety step for release builds (for example typed confirmation phrase).

### 16.3 Async file browser requirement

Document browser must:

- load directory contents on background queue,
- update UI on main actor,
- handle large folders without blocking UI,
- provide explicit error alert if delete/load fails.

### 16.4 Localization completeness

All developer UI strings (titles, rows, confirmations, errors, empty states) must use localization keys.

### 16.5 Metadata/file consistency warning

When deleting files from raw storage browser:

- show warning that this bypasses DB metadata,
- optionally provide a safer delete path that also removes corresponding DB row (if mapping exists).

### 16.6 Analytics hygiene

Developer tool actions should either:

1. not be sent to production analytics, or
2. be tagged with `developer_mode=true` and excluded from business dashboards.

### 16.7 Baseline automated tests

Add tests in every app for:

- manager activation/disable/alert behavior,
- operation guards for destructive actions,
- migration reset action availability rules.

---

## 17) Recommended Developer Diagnostics View (ShareExtension Apps)

For apps with ShareExtension, add a compact diagnostics screen in Developer Mode that shows:

1. build environment (`debug`/`release`)
2. app bundle ID
3. extension bundle ID (configured value)
4. app group identifier
5. shared DB path + existence status
6. shared documents path + existence status
7. migration completion flag
8. legacy DB existence

Optional actions:

- copy diagnostics to clipboard
- re-run migration check (non-destructive)

This significantly reduces time spent debugging environment/capability mismatch.
