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
