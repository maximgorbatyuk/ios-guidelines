# Appearance (Light/Dark/System) Guideline for Future iOS Apps

This guideline is based on analysis of current implementations in:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It defines a reusable standard for implementing app appearance mode (System/Light/Dark), persisting it safely, and keeping it compatible with ShareExtension architecture.

---

## 1) What Current Apps Do

### 1.1 Journey Wallet

- Uses `AppColorScheme` enum (`system`, `light`, `dark`) in `BusinessLogic/Models/UserSettings.swift`.
- Persists value in SQLite `user_settings` key `color_scheme` via `UserSettingsRepository`.
- Uses `ColorSchemeManager` singleton with `@Published currentScheme`.
- App root (`JourneyWalletApp`) applies `.preferredColorScheme(colorSchemeManager.preferredColorScheme)`.
- Settings picker writes through `UserSettingsViewModel.saveColorScheme` -> `ColorSchemeManager.setScheme`.
- Backup/export currently does **not** include color scheme (only currency + language).

### 1.2 CreativityHub

- Uses `AppColorScheme` enum with `colorScheme: ColorScheme?` computed property.
- Persists value in SQLite `user_settings` key `color_scheme`.
- App root (`CreativityHubApp`) keeps local `@State colorScheme`, loads from DB on appear, applies `.preferredColorScheme(colorScheme.colorScheme)`.
- Settings saves via repository and broadcasts `Notification.Name.appColorSchemeDidChange`.
- App root listens to notification and updates theme.
- Backup/export/import **includes** `preferredColorScheme` in `ExportUserSettings`.

### 1.3 EVChargingTracker

- Uses `AppearanceMode` enum (`system`, `light`, `dark`) in `BusinessLogic/Models/UserSettings.swift`.
- Persists value in **UserDefaults** key `appearance_mode` via `AppearanceManager`.
- Does **not** store appearance in SQLite `user_settings` table.
- App root (`EVChargingTrackerApp`) observes `AppearanceManager` and applies `.preferredColorScheme(appearanceManager.colorScheme)`.
- Backup/export/import currently does **not** include appearance mode.

---

## 2) Cross-App Comparison

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| Enum type | `AppColorScheme` | `AppColorScheme` | `AppearanceMode` | Unify naming to `AppColorScheme` |
| Storage backend | SQLite `user_settings` | SQLite `user_settings` | UserDefaults | Prefer SQLite `user_settings` |
| DB key | `color_scheme` | `color_scheme` | none | `color_scheme` |
| Runtime manager | `ColorSchemeManager` observable | Notification + app root state | `AppearanceManager` observable | Single observable manager + optional notification |
| App root apply | direct from manager | `@State` + NotificationCenter | direct from manager | direct observable manager binding |
| Localization keys | namespaced (`settings.color_scheme.*`) | namespaced (`settings.color_scheme.*`) | generic (`"Appearance"`, `"System"`) | namespaced keys everywhere |
| Backup integration | missing color scheme | includes color scheme | missing color scheme | include color scheme in export/import |
| ShareExtension readiness | good (DB/App Group) | good (DB/App Group) | weaker (UserDefaults-only appearance) | DB/App Group-backed preference |

---

## 3) How Value Is Stored (Current State)

### SQLite-based apps (Journey Wallet, CreativityHub)

- Table: `user_settings`
- Key: `color_scheme`
- Value: one of `system`, `light`, `dark`

Repository pattern:

1. `fetchColorScheme()` -> parse raw string to enum, default `.system`
2. `upsertColorScheme(_ scheme)` -> upsert `scheme.rawValue`

### EVChargingTracker

- Storage: `UserDefaults.standard`
- Key: `appearance_mode`
- Value: one of `system`, `light`, `dark`

Important difference:

- value is not visible in `user_settings` table viewer,
- value is not included in DB export/import payload,
- value is not automatically shared with extension process unless suite-based defaults are used.

---

## 4) Gaps Found in Current Implementations

### Gap 1: Storage inconsistency across apps

- EV uses UserDefaults while other apps use SQLite.
- Makes behavior inconsistent for backup/restore and shared container data model.

### Gap 2: Missing backup portability

- Journey Wallet and EV export/import models omit appearance preference.
- Result: restored user may lose chosen theme preference.

### Gap 3: Localization inconsistency

- EV uses generic keys (`"Appearance"`, `"System"`, `"Light"`, `"Dark"`) rather than namespaced settings keys.

### Gap 4: Potential first-frame mismatch in state-driven app root pattern

- CreativityHub loads color scheme in `onAppear`, starting from `.system` state.
- Can cause one-frame mismatch/flicker for users with forced dark/light preference.

### Gap 5: Settings table uniqueness not enforced in some repositories

- Journey and EV repositories create `user_settings` without unique key constraint.
- Upsert still works by query/update, but schema-level uniqueness is safer.

### Gap 6: No dedicated appearance unit tests

- No explicit tests for scheme fetch/upsert/apply behavior across apps.

---

## 5) Canonical Architecture for New Apps

Use this architecture for future apps:

1. `AppColorScheme` enum in shared model layer.
2. `UserSettingsRepository.fetchColorScheme()/upsertColorScheme()` in SQLite.
3. `ColorSchemeManager` singleton (`@MainActor`, `ObservableObject`) as runtime source of truth.
4. App root binds directly:
   - `.preferredColorScheme(colorSchemeManager.preferredColorScheme)`
5. Settings picker updates through manager (which persists to repository).
6. Include color scheme in export/import user settings payload.

### Suggested enum contract

```swift
enum AppColorScheme: String, CaseIterable, Codable {
    case system
    case light
    case dark

    var colorScheme: ColorScheme? {
        switch self {
        case .system: return nil
        case .light: return .light
        case .dark: return .dark
        }
    }
}
```

### Suggested manager contract

```swift
@MainActor
final class ColorSchemeManager: ObservableObject {
    @Published private(set) var currentScheme: AppColorScheme

    func setScheme(_ scheme: AppColorScheme)
    var preferredColorScheme: ColorScheme? { currentScheme.colorScheme }
}
```

---

## 6) Persistence Rules (Must Follow)

1. Store in SQLite `user_settings` as `color_scheme`.
2. Default to `.system` when setting is absent or invalid.
3. Use single-source upsert path (repository method only).
4. Prefer unique key constraint for `user_settings.key` in schema.
5. Avoid mixing UserDefaults + SQLite for the same preference.

---

## 7) Settings UI Rules

1. Keep picker in Settings > Preferences/Base Settings.
2. Use namespaced localization keys:
   - `settings.color_scheme`
   - `settings.color_scheme.system`
   - `settings.color_scheme.light`
   - `settings.color_scheme.dark`
3. Emit analytics event on change (optional but recommended).
4. Apply new scheme immediately in current session.

---

## 8) Startup and First-Frame Behavior

To avoid flicker/mismatch:

1. Initialize runtime color scheme before first main frame if possible.
2. If loading from DB, ensure DB is ready before manager first read.
3. Prefer manager-bound app root over delayed `onAppear` assignment when feasible.

For apps with App Group DB migration:

- run migration before first DB connection,
- warm up `DatabaseManager.shared` early if needed,
- then read appearance value.

---

## 9) Backup / Export / Import Requirements

Appearance preference should be portable.

Required:

1. Include `preferredColorScheme` in export `userSettings` payload.
2. Restore it during import via repository upsert.
3. Apply restored scheme in active session after import (manager update or notification).

Do not silently drop appearance preference during restore.

---

## 10) ShareExtension Considerations

Appearance feature is app-focused, but ShareExtension-compatible apps should follow these rules:

1. Keep preference in shared App Group DB (`user_settings`) for consistency.
2. If extension UI later adopts custom theme behavior, it can read same `color_scheme` value.
3. Do not rely on process-local UserDefaults for cross-target consistency.
4. Ensure migration to App Group DB happens before first settings read in app startup.
5. Appearance logic must not block ShareExtension startup flow.

---

## 11) Apple Developer Website Steps

### 11.1 For appearance feature alone

No special Apple Developer capability is required for light/dark/system switching itself.

### 11.2 For apps that also include ShareExtension (recommended alignment)

Because appearance preference should live in shared DB/App Group for consistency:

1. Create App Groups in Apple Developer portal:
   - production: `group.<team>.<app>`
   - development: `group.<team>.<app>.dev`
2. Create/verify App IDs for:
   - app release/debug,
   - extension release/debug.
3. Enable App Groups capability on all relevant App IDs.
4. Regenerate provisioning profiles after capability updates.
5. Verify entitlements and `Info.plist` app group identifiers match exactly.

### 11.3 App Store Connect

1. Mention appearance improvements in release notes when changed.
2. Validate screenshots in both light/dark if app visuals differ significantly.

---

## 12) Test Matrix

### Repository tests

1. `fetchColorScheme()` default is `.system` when empty.
2. upsert + fetch roundtrip for all values.
3. invalid stored value falls back safely to `.system`.

### Manager tests

1. initial load from repository.
2. `setScheme` updates published state.
3. persistence failure behavior is handled (logged/reported) deterministically.

### UI tests

1. settings picker shows all three options.
2. selecting each option applies app theme immediately.
3. selected value persists after app restart.

### Backup/import tests

1. export contains `preferredColorScheme`.
2. import restores same scheme.
3. restored scheme applies in-session without restart.

### ShareExtension apps

1. upgrade migration scenario preserves appearance preference.
2. extension-first launch does not corrupt settings storage.

---

## 13) Definition of Done

Appearance implementation is complete only if:

1. user can choose System/Light/Dark in settings,
2. value is persisted in SQLite `user_settings` as `color_scheme`,
3. app root applies scheme consistently across launches,
4. backup/export/import preserves preference,
5. all appearance strings are localized with namespaced keys,
6. tests cover repository + manager + restore flow,
7. ShareExtension-enabled apps keep storage consistent in App Group context.

---

## 14) Recommended Defaults for New Apps

Use these defaults unless product requirements differ:

- enum name: `AppColorScheme`
- values: `system`, `light`, `dark`
- storage: SQLite `user_settings.color_scheme`
- runtime owner: `ColorSchemeManager` observable singleton
- app root apply: `.preferredColorScheme(manager.preferredColorScheme)`
- localization keys: `settings.color_scheme*`
- export/import: include `preferredColorScheme`
- ShareExtension apps: keep setting in App Group DB, not process-local defaults

---

## 15) Gaps Found in Second Analysis Pass

After a deeper re-check, these additional gaps were identified.

### Gap A: Runtime theme may not refresh after import

- In state + notification-based patterns (like current CreativityHub), import updates DB but may not broadcast a color-scheme-change notification.
- Risk: restored scheme is persisted but not applied immediately until restart.

### Gap B: Non-atomic persistence/apply sequence

- Some flows update in-memory/apply state before confirming DB persistence succeeded.
- Risk: UI shows new theme while stored value remains old (state drift after restart).

### Gap C: Missing `@MainActor` boundary on appearance managers

- Managers that publish UI-observed state should be isolated to main actor.
- Risk: accidental cross-thread state changes.

### Gap D: EV storage not represented in repository protocol

- EV currently stores appearance outside `UserSettingsRepository`.
- Risk: inconsistent app architecture, no table visibility, no export portability.

### Gap E: No migration path from legacy UserDefaults appearance

- If app migrates from UserDefaults to DB-backed `color_scheme`, legacy value may be lost without one-time migration logic.

### Gap F: No dedicated diagnostics for stored-vs-applied scheme

- Developer tools currently do not consistently show both:
  - persisted value in storage,
  - currently applied runtime scheme.
- Risk: harder debugging of "saved but not applied" bugs.

---

## 16) Rules to Close Second-Pass Gaps

### 16.1 Import must apply scheme in-session

After import restores `preferredColorScheme`:

1. update repository value,
2. update runtime manager state (or emit app-wide notification),
3. verify app root re-renders with restored scheme without restart.

### 16.2 Persistence-before-publish rule

For `setScheme`/`saveColorScheme`:

1. persist first (or persist + rollback strategy),
2. only then publish/apply runtime state,
3. on failure: keep old runtime value and surface log/error.

### 16.3 Main-actor isolation

Mark appearance manager APIs as `@MainActor` when they mutate `@Published` UI state.

### 16.4 Unified repository contract

Future apps should include appearance APIs in repository protocol:

- `fetchColorScheme() -> AppColorScheme`
- `upsertColorScheme(_ scheme: AppColorScheme) -> Bool`

### 16.5 One-time legacy migration (if needed)

When moving from UserDefaults (`appearance_mode`) to DB (`color_scheme`):

1. on first launch after update, read legacy value,
2. if DB value absent, write migrated value to DB,
3. set migration marker,
4. optionally keep legacy key for one release then clean up.

### 16.6 Add developer diagnostics row

In Developer Mode, show:

- stored appearance value (DB key/raw),
- active runtime value,
- effective applied `ColorScheme` (`nil`, `light`, `dark`).

---

## 17) Additional Test Cases (Second-Pass Additions)

Add these to the test matrix:

1. Import restores theme and applies immediately (no restart).
2. Persistence failure does not change runtime scheme.
3. Legacy UserDefaults-to-DB migration runs once and preserves user choice.
4. Manager mutation happens on main actor only.
