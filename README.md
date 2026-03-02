# iOS Guidelines

Reusable implementation standards derived from cross-app analysis of:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

## Documents

- `ios-guidelines/share-extension-guideline.md` - App Group architecture, parser priority, migration safety, extension target boundaries, Apple Portal/App Store setup.
- `ios-guidelines/developer-mode-guideline.md` - Hidden activation flow (15 taps), manager pattern, safe tooling gates, testing strategy.
- `ios-guidelines/app-version-checker-guideline.md` - App Store version lookup, comparison strategy, badge behavior, release/testing checklist.
- `ios-guidelines/appearance-guideline.md` - Light/dark/system architecture, persistence, migration, import/export consistency, test matrix.
- `ios-guidelines/onboarding-guideline.md` - Launch gating, onboarding structure/content, completion state, restart behavior.
- `ios-guidelines/analytics-guideline.md` - Firebase/GA setup, Debug vs Release telemetry rules, `user_id` handling, privacy/compliance checks.
- `ios-guidelines/potential-issue-fixes.md` - Cross-feature preventive fixes for configuration drift, entitlement tokenization, and release hardening.

## Suggested Reading Order

1. `ios-guidelines/share-extension-guideline.md`
2. `ios-guidelines/developer-mode-guideline.md`
3. `ios-guidelines/onboarding-guideline.md`
4. `ios-guidelines/appearance-guideline.md`
5. `ios-guidelines/app-version-checker-guideline.md`
6. `ios-guidelines/analytics-guideline.md`
7. `ios-guidelines/potential-issue-fixes.md`

## Current Status

- All seven guideline documents are present.
- `ios-guidelines/potential-issue-fixes.md` is the default place for cross-feature preventive standards.
