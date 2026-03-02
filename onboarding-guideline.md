# Onboarding Guideline for Future iOS Apps

This guideline is based on a comparative analysis of onboarding implementations in:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It is intended to be a practical implementation standard for new apps.

---

## 1) What Current Apps Do

## 1.1 Entry flow in all three apps

All apps follow this root flow:

`LaunchScreen -> Onboarding (if needed) -> MainTabView`

Common behavior:

1. Onboarding completion flag is stored in `UserDefaults` via `@AppStorage`.
2. Flag key is consistently `isOnboardingComplete`.
3. If flag is `false`, app shows onboarding on startup.
4. If flag is `true`, app skips onboarding.

## 1.2 App-specific onboarding content

### Journey Wallet

- Language selection first page.
- Feature pages include:
  - journeys,
  - bookings/documents,
  - reminders,
  - checklists,
  - roadmap,
  - statistics.
- Uses custom page indicator (capsule + dots).
- Uses per-page gradient backgrounds.

### EVChargingTracker

- Language selection first page.
- Feature pages include:
  - charging tracking,
  - cost monitoring,
  - maintenance,
  - documents,
  - ideas,
  - stats/CO2 insights.
- Uses custom page indicator (same style family as Journey Wallet).
- Uses per-page gradient backgrounds.

### CreativityHub

- Language selection first page.
- Feature pages include:
  - welcome/value,
  - projects,
  - ideas,
  - budget,
  - work logs.
- Uses default page indicator (`TabView` with page style + index).
- Uses per-page gradient backgrounds.

---

## 2) How Onboarding Is Reopened (Current State)

Important finding:

- In all three apps, onboarding restart is currently triggered from **Support/About settings area**, not from a dedicated Developer Mode section.

Current mechanism in each app:

1. User taps "Start onboarding again" in settings.
2. App clears onboarding flag:
   - `UserDefaults.standard.removeObject(forKey: "isOnboardingComplete")`
3. Root view logic naturally shows onboarding next time relevant root state is evaluated.

Developer mode relation:

- Restart onboarding does **not** require developer mode in current implementations.

---

## 3) Cross-App Architecture Comparison

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| View model type | `ObservableObject` | `@Observable` | `ObservableObject` | Use project-wide standard (`@Observable` if app uses Observation) |
| Onboarding pages model | stores already localized strings | stores localization keys | stores already localized strings | Prefer key-based page model |
| Language update strategy | change language + rebuild pages | change language; render keys dynamically | change language + rebuild pages | Prefer dynamic key rendering (no page rebuild required) |
| Completion write location | root callbacks set `@AppStorage` + `UserDefaults` | onboarding VM sets completion + callback updates state | root callbacks set `@AppStorage` + `UserDefaults` | Single owner for completion side effect |
| Restart onboarding location | settings support section | settings about section | settings support section | Keep in support + optional developer shortcut |
| Analytics coverage | rich onboarding events | minimal onboarding analytics | rich onboarding events | Define consistent event set |
| Localization key style | mixed (generic keys like "Next") | namespaced (`onboarding.*`) | mixed (generic keys like "Next") | Use namespaced keys everywhere |

---

## 4) Standard Onboarding Contract for New Apps

Implement onboarding with these components:

1. `OnboardingViewModel`
2. `OnboardingView`
3. `OnboardingLanguageSelectionView`
4. `OnboardingPageView`
5. `OnboardingPageItem` model (key-based)
6. root gating in app root/content root using `@AppStorage`

### Required state contract

- `onboardingCompletedKey = "isOnboardingComplete"`
- `currentPage` starts at `0` (language page)
- `totalPages = 1 + featurePages.count`

---

## 5) Page Content Strategy

### Recommended page sequence

1. Language selection
2. Core value proposition
3. 3-6 domain feature pages
4. Final "Get started"

### Feature page content rules

- Each page should communicate one user benefit.
- Avoid technical implementation text.
- Keep title + subtitle concise.
- Use icon + accent color tied to domain meaning.

### Page model recommendation

Prefer storing localization keys:

```swift
struct OnboardingPageItem: Identifiable {
    let id: Int
    let icon: String
    let titleKey: String
    let descriptionKey: String
    let color: Color
}
```

This avoids recreating page arrays when language changes.

---

## 6) Navigation and Controls

Required controls:

1. `Next` button on intermediate pages.
2. `Get started` button on last page.
3. `Skip` button available after language page.

Behavior rules:

- `Skip` marks onboarding as complete immediately.
- `Get started` marks onboarding as complete.
- Language page uses explicit continue action.

---

## 7) Visual Pattern Standard

Recommended default (already used across apps):

1. Full-screen gradient background in `ZStack`.
2. Language page uses neutral blue/cyan gradient.
3. Feature pages use page-specific accent gradient (`0.3` + `0.1` opacity stops).
4. Transition with short fade (`.transition(.opacity)`).

This preserves cross-app visual consistency while keeping each app distinct via per-page colors/icons.

---

## 8) Localization Standard

### Must-have rules

1. No hardcoded onboarding strings.
2. Use namespaced keys:
   - `onboarding.language.title`
   - `onboarding.language.description`
   - `onboarding.skip`
   - `onboarding.next`
   - `onboarding.get_started`
   - `onboarding.<feature>.title`
   - `onboarding.<feature>.description`
3. Keep onboarding keys complete for all supported languages.

### Gap found in current apps

- Journey Wallet and EV still use several generic keys (`"Next"`, `"Skip"`, `"Get started"`, `"Welcome to"`, `"Select your language"`).

Recommendation:

- Migrate to namespaced `onboarding.*` keys (CreativityHub style).

---

## 9) Restart Onboarding (User + Developer Workflows)

### User-facing restart (required)

Keep a visible settings action in Support/About section:

- label: `settings.start_onboarding_again`
- action: clear completion flag

### Developer-mode shortcut (recommended)

Because QA often needs repeated onboarding checks, add a Developer Mode action too:

1. "Reset onboarding state"
2. Clears same completion flag
3. Optionally navigates immediately to onboarding root

This keeps current user-facing behavior while improving QA speed.

---

## 10) Completion Ownership and Side Effects

Current apps use mixed ownership (root closures vs view model completion write).

Recommended standard:

1. Choose one owner for completion write (prefer ViewModel or coordinator).
2. Root should only react to `@AppStorage` state, not duplicate persistence logic.
3. Avoid writing same key from multiple layers unless intentional.

---

## 11) Analytics Standard (Recommended)

Use a consistent minimal event set:

1. `onboarding_screen_viewed`
2. `onboarding_language_selected`
3. `onboarding_skip_tapped`
4. `onboarding_next_tapped`
5. `onboarding_completed`
6. `onboarding_restart_tapped`

Keep event properties stable:

- `screen`
- `page_index`
- `language`

---

## 12) ShareExtension Compatibility Notes

Onboarding itself is app-only, but new apps with ShareExtension should follow these rules:

1. Onboarding views must remain main-app-target only.
2. ShareExtension must never attempt to present onboarding.
3. Restart-onboarding action must not interfere with App Group migration logic.
4. If onboarding changes language or settings storage, ensure extension-safe localization access remains valid.

---

## 13) Test Matrix

### Flow tests

1. First launch shows onboarding.
2. Completing onboarding hides it on next launch.
3. Skipping onboarding hides it on next launch.
4. Restart action re-enables onboarding flow.

### Language tests

1. Language selection updates onboarding text immediately.
2. Selected language persists after onboarding completion.

### UI tests

1. Page count and order are correct.
2. Skip button visibility rule works (hidden on language page and last page).
3. Final button text switches to `Get started` only on last page.

### Regression tests

1. Launch screen transition to onboarding/main works smoothly.
2. No onboarding flash after completion flag is true.
3. Localization keys exist in all supported locales.

---

## 14) Definition of Done

Onboarding implementation is complete only if:

1. startup gating works (`LaunchScreen -> Onboarding/Main`),
2. onboarding can be completed/skipped,
3. language selection works and persists,
4. restart onboarding action exists in settings,
5. onboarding copy is fully localized with namespaced keys,
6. analytics events are emitted consistently,
7. onboarding remains isolated from ShareExtension runtime.

---

## 15) Recommended Defaults for New Apps

Use these defaults unless requirements differ:

- key-based onboarding page model (`onboarding.*` keys)
- language-first onboarding page
- 4-6 feature pages based on core user value
- per-page gradient background style
- settings action `settings.start_onboarding_again`
- optional duplicate reset action in Developer Mode for QA
- unified completion ownership in one layer
