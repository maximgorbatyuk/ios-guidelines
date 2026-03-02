# Potential Issue Fixes Guideline for Future iOS Apps

This document captures cross-feature preventive fixes that reduce configuration drift and release risk.

---

## 1) App Group Entitlements Must Be Build-Setting Driven

### Why this matters

Hardcoding App Group IDs inside entitlements can drift from project configuration (`xcconfig`) values over time. That drift can cause:

- app and extension losing access to the shared container
- signing/provisioning mismatches
- Debug/Release environment mixups

### Standard

Use the build setting token in entitlements instead of literal App Group strings:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>$(APP_GROUP_IDENTIFIER)</string>
</array>
```

Never hardcode values like `group.dev...` directly in entitlement files.

### Required configuration

1. Define `APP_GROUP_IDENTIFIER` in `Config/Base.xcconfig`.
2. Override by configuration only when needed (for example, a Debug-only suffix).
3. Ensure Release inherits from Base when no override is required.
4. Keep `Info.plist` App Group references aligned with `$(APP_GROUP_IDENTIFIER)`.

### Verification checklist

1. Confirm `com.apple.security.application-groups` in entitlements uses `$(APP_GROUP_IDENTIFIER)`.
2. Confirm `APP_GROUP_IDENTIFIER` resolves for Debug and Release via `xcodebuild -showBuildSettings`.
3. Confirm app and extension targets both use xcconfig-backed build configurations.
4. Validate entitlements syntax with `plutil -lint`.

### Scope

Apply this rule to all app and extension targets in this workspace unless a target has a documented exception.

---

## 2) Xcode Cloud Signing Metadata Must Exist in TargetAttributes

### Why this matters

Targets with entitlements (app + extensions) require a valid signing context during archive/export. If project-level target signing metadata is missing, CI can fail in `exportArchive` with exit code 70 even when local simulator builds pass.

Common symptom patterns:

- `exportArchive` fails with exit code 70 in Xcode Cloud
- target error: `has entitlements that require signing with a development certificate`
- archive succeeds in some local flows but export fails in CI

### Standard

In `*.xcodeproj/project.pbxproj`, under `PBXProject -> attributes -> TargetAttributes`, ensure every entitlement-bearing target contains:

- `DevelopmentTeam = <TEAM_ID>;`
- `ProvisioningStyle = Automatic;`

Apply this to the main app target and any embedded extension targets (for example Share Extension).

### Verification checklist

1. Confirm both keys are present in `TargetAttributes` for all entitlement-bearing targets.
2. Run release archive with automatic provisioning updates:
   - `xcodebuild -project <Project>.xcodeproj -scheme <Scheme> -configuration Release -destination 'generic/platform=iOS' -allowProvisioningUpdates archive`
3. Run export from the archive using the expected distribution method for your pipeline.
4. If CI shows entitlement-related signing errors, check `TargetAttributes` before changing export flags or forcing ad-hoc signing overrides.

### Scope

Apply this rule to all workspace apps and extensions that include entitlements.
