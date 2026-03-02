# Share Extension Guideline for Future iOS Apps

This guideline is based on a comparative analysis of:

- `journey-wallet`
- `CreativityHub`
- `EVChargingTracker`

It is intended to be a reusable implementation standard for future apps.

---

## 1) What Works Best Across Existing Apps

### Common strong patterns

1. **Single ShareExtension target + xcconfig-driven Debug/Release values**
   - All apps use one extension target.
   - `$(APP_GROUP_IDENTIFIER)` and `$(SHARE_EXTENSION_BUNDLE_ID)` are supplied from xcconfig.
   - This avoids duplicate target maintenance.

2. **Shared data through App Group container**
   - Main app and extension use the same SQLite and document directory in App Group.
   - `AppGroupIdentifier` is present in both app and extension `Info.plist`.

3. **Normalized parser model -> form ViewModel -> repository save**
   - Parser normalizes `NSExtensionItem` input.
   - ViewModel pre-fills fields and validates form.
   - Save happens through existing repositories and services.

4. **Defensive file lifecycle**
   - Copy temporary provider files before extension closes.
   - Clean up temporary files on cancel/success/failure.

5. **Extension is offline-first and local-only**
   - No server dependency required to save shared content.

---

## 2) Comparative Analysis (Current Apps)

| Aspect | Journey Wallet | CreativityHub | EVChargingTracker | Recommendation |
|---|---|---|---|---|
| Parser style | `DispatchGroup` + callbacks | Async parser service (`InputParser`) | Async parser, early-return | Prefer Creativity async parser style |
| Content priority | **files > url > text** | **image > file > url > text** | **url > text > image > file** | Standardize to **file/image > url > text** |
| Multiple files | Supported (up to 10) | Activation allows 10, parser currently returns one best match | Parser returns first match | If app needs docs/media import, support multi-file batch |
| Migration safety | One-time migration exists, but legacy cleanup strategy is permissive | No legacy migration needed (App Group-first storage) | Strict migration helper + validation + extension gate | Use EV strict migration approach |
| Localization strategy | Share keys in app strings; docs emphasize lproj membership | Dedicated ShareExtension lproj + app lproj | App lproj + extension-aware bundle resolution in `LocalizationManager` | Use EV bundle-resolution + strict key naming |
| File save rollback | Partial (can leave file when DB insert fails) | Strong rollback on insert failure | Strong rollback on insert failure | Always rollback file if DB insert fails |
| Extension unavailable APIs | Implicitly handled by target setup | Clean target split by folders | Explicit `pbxproj` exceptions for restricted services | Always enforce extension-safe membership |

---

## 3) Canonical Architecture for New Apps

Implement Share Extension using this structure:

1. `ShareViewController` (UIKit entry point)
2. `InputParser` (async extraction + normalization)
3. `SharedInput` model (`kind`, url/text/files metadata)
4. `ShareFormViewModel` (`@MainActor`, validation + save)
5. `ShareFormView` (`NavigationStack`, localized form)
6. Existing repositories/services for persistence (`DatabaseManager.shared`)

### Required data flow

`NSExtensionItem -> InputParser -> SharedInput -> ShareFormViewModel -> Repository/DocumentService -> Complete Request`

---

## 4) Content Parsing Rules (Priority + Fidelity)

### Priority rule (recommended default)

1. **Image/File attachments**
2. **URL**
3. **Plain text**

Reason:

- In real share sheets, URL + image/text often arrive together.
- If file/image is present, users usually intend to save the attachment, not only the URL.

### Parser requirements

- Parse all attachments first, then decide by priority (do not early-return too soon).
- Preserve both URL and snippet when both exist.
- Use `attributedContentText` as suggested title/snippet when available.
- Detect URL inside plain text via `NSDataDetector`.
- Keep deterministic behavior for mixed payloads.

### Type extraction rules

- Prefer `loadFileRepresentation` for files/images to avoid memory spikes.
- Avoid loading large files into `Data` too early.
- If `loadItem` returns `URL`, copy file immediately to extension temp.

---

## 5) Coding Style Standard (for Share Extension)

- Use `final class` for services/view models where possible.
- Use `@MainActor` on form ViewModel.
- Use `os.Logger`; do not use `print()`.
- Use `NavigationStack` (not `NavigationView`).
- Use explicit `SwiftUI.View` in projects that `_exported import SQLite` (avoids `View` ambiguity with `SQLite.View`).
- Keep extension code split into `Models`, `Services`, `UI`, `ViewModel`.
- Keep Save actions idempotent and guarded by validation.

---

## 6) Localization Standard

### Must-have rules

1. No hardcoded user-facing strings.
2. Use namespaced keys:
   - `share.title`
   - `share.section.content`
   - `share.error.save_failed`
3. Avoid generic keys like `"Title"`, `"URL"`, `"Description"` for new Share UI.
4. Ensure extension can resolve localization resources in app-extension context.

### Recommended implementation

- Keep one global `L()` API.
- In extension, resolve bundle safely:
  - if `Bundle.main` is `.appex`, optionally resolve containing `.app` bundle for `.lproj` files.
- Alternatively/additionally include extension-localized resources (`ShareExtension/*.lproj`).

### Validation checklist

- Every Share key exists in all supported languages.
- Key lookup never falls back to raw key in extension UI.

---

## 7) App Group + Database Strategy

### Required setup

- Main app and extension must use same App Group per build configuration.
- DB path and document path must be App Group-based.

### Migration standard (use EV strict pattern)

1. Run migration **before opening DB connection** in `DatabaseManager` init.
2. Use shared migration marker in `UserDefaults(suiteName: appGroup)`, not local defaults.
3. Migrate via: copy to temp -> validate -> atomic move -> cleanup legacy.
4. Include SQLite sidecars (`-wal`, `-shm`).
5. Do not delete legacy DB unless destination is validated.

### Extension-first safety

- If migration is not complete and shared DB is not ready:
  - show blocking localized message,
  - instruct user to open main app once,
  - do not create empty shared DB.

---

## 8) Document/File Handling Standard

### Storage

- Persist files under App Group documents directory (entity-scoped subfolders).
- Store enough metadata for preview and export/import:
  - custom title
  - fileName
  - fileType
  - size
  - createdAt/updatedAt
  - stable file locator (prefer relative path or deterministic path derivation)

### Save transaction behavior

1. Persist file
2. Insert DB record
3. If DB insert fails, delete persisted file (rollback)

### Cleanup

- Remove temp files when extension closes/cancels/fails.
- On entity delete, remove file + DB row.
- On wipe/import replacement, clear both DB metadata and file storage.

---

## 9) Backup / Export / Import Rules (Critical)

Document support in Share Extension is not complete unless backup round-trip works.

### Must-have

1. Export/import must keep documents openable after restore.
2. Do not restore metadata-only docs as normal documents.
3. Import must fail on document/idea insertion failures (no silent partial success).
4. Wipe/import flow must not leave orphaned files.

### Common failure to avoid

- Restoring document rows with `filePath=nil` or invalid path and still reporting success.

---

## 10) Frequent Issues Seen and How to Prevent Them

1. **App Group mismatch across targets/configurations**
   - Symptom: extension cannot access DB/documents.
   - Prevention: explicit per-config matrix of bundle IDs, entitlements, app groups.

2. **Parser priority mismatch with user intent**
   - Symptom: sharing image+url saves only URL.
   - Prevention: parse all candidates and apply deterministic priority (`file/image > url > text`).

3. **Activation rule mismatch with parser support**
   - Symptom: parser supports files but extension never appears for files.
   - Prevention: keep `NSExtensionActivationRule` aligned with parser capabilities.

4. **Extension-unavailable APIs compiled into extension target**
   - Symptom: build errors or runtime restrictions.
   - Prevention: exclude restricted files (`Firebase`, `BGTaskScheduler`, etc.) from extension target membership.

5. **Localization missing inside extension**
   - Symptom: UI shows raw localization keys.
   - Prevention: extension-aware bundle lookup and verified key coverage per locale.

6. **Unsafe migration when extension launches first**
   - Symptom: empty shared DB creation / legacy data loss risk.
   - Prevention: strict migration helper + extension gate + shared migration marker.

7. **File/DB inconsistency on save failure**
   - Symptom: orphan files in storage.
   - Prevention: transactional save with rollback.

8. **Backup/import document corruption**
   - Symptom: restored docs listed but not previewable.
   - Prevention: include restorable file references + file payload strategy + strict import validation.

9. **Memory spikes for large attachments**
   - Symptom: extension killed by system.
   - Prevention: prefer file URL copy pipeline over `Data(contentsOf:)` for large files.

10. **SwiftUI `View` ambiguity with SQLite**
   - Symptom: compile error (`View is ambiguous`).
   - Prevention: use explicit `SwiftUI.View` in extension UI files.

---

## 11) Apple Developer Portal Steps (Manual)

Do these steps before testing/signing on devices/TestFlight.

### A. Create App Groups

Apple Developer -> Certificates, Identifiers & Profiles -> Identifiers -> **App Groups** -> `+`

Create:

- Production group: `group.<team>.<app>`
- Development group: `group.<team>.<app>.dev`

### B. Create/verify App IDs

Apple Developer -> Identifiers -> **App IDs**

Create/verify:

1. Main app Release bundle ID
2. Main app Debug bundle ID (if explicit debug app ID is used)
3. ShareExtension Release bundle ID
4. ShareExtension Debug bundle ID (if explicit debug extension ID is used)

### C. Enable App Groups capability on all relevant App IDs

Attach groups as follows (recommended strict mapping):

- Main app Release -> prod group
- Main app Debug -> dev group
- ShareExtension Release -> prod group
- ShareExtension Debug -> dev group

Optional broader mapping (allowed but less strict): attach both groups to both main app IDs.

### D. Provisioning profiles

After capability changes, regenerate profiles:

- iOS App Development profiles (app + extension)
- App Store distribution profiles (app + extension)

Then refresh signing in Xcode.

---

## 12) Xcode Project Configuration Checklist

1. Add `ShareExtension` target.
2. Add to main app build phase: **Embed Foundation Extensions**.
3. `PRODUCT_BUNDLE_IDENTIFIER = $(SHARE_EXTENSION_BUNDLE_ID)`.
4. Set `CODE_SIGN_ENTITLEMENTS = ShareExtension/ShareExtension.entitlements` for both Debug and Release (a single file using `$(APP_GROUP_IDENTIFIER)` resolves correctly per configuration via xcconfig).
5. Add `AppGroupIdentifier = $(APP_GROUP_IDENTIFIER)` to both app and extension `Info.plist`.
6. Keep app + extension using same xcconfig variable source.
7. Ensure extension target includes only extension-safe code.
8. Verify `NSExtensionActivationRule` matches supported content types.

---

## 13) App Store Connect Steps

1. No separate listing is needed for extension; it ships inside main app.
2. Update release notes to mention Share Extension capability.
3. Update App Privacy answers if extension changes data handling.
4. Add App Review Notes with exact validation steps:
   - source app/content,
   - expected Share sheet behavior,
   - expected result in target app,
   - migration note if app must be opened once after update.
5. Validate in TestFlight on both:
   - clean install
   - upgrade install from previous production build

---

## 14) Test Matrix (Required Before Release)

### Content type matrix

- URL only
- Text only
- URL + text
- Single image
- Single PDF/file
- Multiple files/images (if supported)

### Source apps

- Safari
- Files
- Photos
- Mail
- Notes

### Migration matrix

- Fresh install
- Upgrade with legacy DB
- Extension opened before app after upgrade
- Legacy DB with WAL/SHM present

### Localization matrix

- Validate all supported languages in extension UI

### Data consistency

- DB row created + file physically exists
- Delete removes both row and file
- Backup/export/import keeps documents previewable

### Failure cases

- No target container entities (no journeys/projects/cars)
- Save failure rollback
- Corrupted/unsupported attachment handling

---

## 15) Definition of Done for Share Extension

Feature is done only if all are true:

1. ShareExtension appears in iOS share sheet for configured content types.
2. Saves successfully to correct shared DB and document storage.
3. Migration path is safe and idempotent.
4. Extension-first launch cannot cause data loss.
5. Localization works in all app-supported languages.
6. No extension-unavailable API violations.
7. Backup/import for shared entities is reliable.
8. Debug and Release signing/profiles are valid.

---

## 16) Recommended Default Decisions for New Apps

If a future app has no special constraints, use these defaults:

- **Parser priority:** file/image > url > text
- **Activation rule:** text + web URL + image (10) + file (10)
- **Storage:** App Group container only
- **Migration:** strict, validated, shared marker, extension gate
- **Localization:** namespaced `share.*` keys + extension-aware bundle lookup
- **Save logic:** transactional with rollback and cleanup
- **Testing:** include upgrade + extension-first scenarios, not just clean install

---

## 17) Extension Lifecycle Contract (Gap Coverage)

These rules are critical and were not explicit enough before.

1. Call `extensionContext.completeRequest(returningItems:)` exactly once on success.
2. Call `extensionContext.cancelRequest(withError:)` exactly once on user cancel/failure.
3. Do not leave pending async tasks after close; cancel long-running parser/save tasks in `deinit`/dismiss path.
4. Keep UI responsive while parsing; never block main thread with large file I/O.
5. Treat extension process as ephemeral: persist needed temp outputs immediately.

---

## 18) Activation Rule Template (Gap Coverage)

Keep parser support and `Info.plist` activation in sync.

Recommended baseline:

- `NSExtensionActivationSupportsText = YES`
- `NSExtensionActivationSupportsWebURLWithMaxCount = 1`
- `NSExtensionActivationSupportsImageWithMaxCount = 10`
- `NSExtensionActivationSupportsFileWithMaxCount = 10`

Rules:

- If parser supports multi-file import, max count must reflect that.
- If parser handles only single file, set max count to `1` or show explicit "first file only" behavior.
- Re-test appearance in share sheet after every activation-rule change.

---

## 19) Security and Input Hardening (Gap Coverage)

1. Never trust attachment filename from host app; sanitize and generate internal storage names.
2. Validate allowed file types by `UTType`/extension and reject unsupported executable/script types.
3. Enforce max attachment count and optional file size guardrails to prevent OOM/timeouts.
4. Normalize URLs before save (trim whitespace, ensure valid absolute URL if field requires URL).
5. Escape/sanitize user-provided title/notes for safe display and export serialization.

---

## 20) Early DB Warm-Up for DB-Backed UI State (Gap Coverage)

If app-level UI state (for example color scheme, language override, feature flags) is stored in SQLite:

1. Touch `DatabaseManager.shared` early during app startup.
2. Ensure migration to App Group runs before first DB-backed settings read.
3. Validate first rendered frame already reflects restored settings (no flicker/reset).

This is especially important in upgrade builds where extension migration is introduced.

---

## 21) CI / Verification Commands (Gap Coverage)

Minimum verification before merge/release:

1. Build main app (Debug + Release).
2. Build ShareExtension target (Debug + Release).
3. Run unit tests including parser and save-path tests.
4. Run upgrade migration scenario on simulator/device (legacy DB -> App Group DB).
5. Validate extension appears in share sheet from Safari, Files, and Photos.

Recommended extra tests:

- parser mixed-payload tests (image+url+text)
- transactional rollback tests (file created, DB insert fails)
- localization smoke tests in all supported locales

---

## 22) Apple Portal Final Validation Checklist (Gap Coverage)

Before App Store submission, verify in Apple Developer portal:

1. App Group capability is enabled on **every** relevant App ID (main + extension, debug + release where applicable).
2. App Group IDs in portal exactly match entitlements (character-for-character).
3. Extension App IDs are explicit (not wildcard).
4. Regenerated provisioning profiles are assigned to correct targets/configurations.
5. Signing identity/profile pair in archive includes the extension target and embeds `.appex` correctly.
