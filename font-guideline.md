# Custom Font (JetBrains Mono) Guideline for Future iOS Apps

Based on the implementation in `EVChargingTracker/`. Defines how to ship a custom font with language-aware fallback to the system font.

---

## 1) Goal

- Adopt a monospace display font (JetBrains Mono) across the app for supported languages.
- Fall back to the system font for languages whose scripts the custom font does not cover (in our case: Kazakh `kk`, Simplified Chinese `zh-Hans`).
- Keep the ShareExtension on the system font to avoid bundling duplicate TTFs.
- Stay compatible with Dynamic Type, Light/Dark mode, and runtime language switching.

---

## 2) File Layout

```
<App>/
├── Fonts/
│   ├── JetBrainsMono-Regular.ttf
│   ├── JetBrainsMono-Medium.ttf
│   ├── JetBrainsMono-Bold.ttf
│   ├── JetBrainsMono-Italic.ttf
│   └── JetBrainsMono-BoldItalic.ttf
├── Shared/
│   ├── AppFont.swift            # resolver + .appFont(_:) modifier
│   └── AppFontAppearance.swift  # UIKit chrome integration
└── Info.plist                   # UIAppFonts array
```

Drop the TTFs into the main app's synchronized root group. Do **not** add them to the ShareExtension group. The Fonts/ directory lives only in the main target.

---

## 3) Registering Fonts

Add to `Info.plist`:

```xml
<key>UIAppFonts</key>
<array>
    <string>JetBrainsMono-Regular.ttf</string>
    <string>JetBrainsMono-Medium.ttf</string>
    <string>JetBrainsMono-Bold.ttf</string>
    <string>JetBrainsMono-Italic.ttf</string>
    <string>JetBrainsMono-BoldItalic.ttf</string>
</array>
```

PostScript names:

| File | PostScript name |
|------|-----------------|
| `JetBrainsMono-Regular.ttf` | `JetBrainsMono-Regular` |
| `JetBrainsMono-Medium.ttf` | `JetBrainsMono-Medium` |
| `JetBrainsMono-Bold.ttf` | `JetBrainsMono-Bold` |
| `JetBrainsMono-Italic.ttf` | `JetBrainsMono-Italic` |
| `JetBrainsMono-BoldItalic.ttf` | `JetBrainsMono-BoldItalic` |

---

## 4) `AppFont` Resolver

Lives in `<App>/Shared/AppFont.swift`. Responsibilities:

- Expose `AppFontStyle` enum matching SwiftUI's standard styles (`largeTitle`, `title`, `title2`, `title3`, `headline`, `subheadline`, `body`, `callout`, `footnote`, `caption`, `caption2`) plus `.custom(size:relativeTo:)`.
- Branch on `AppLanguage`: supported → `Font.custom(postScript, size:, relativeTo:)`, unsupported → `Font.system(...)`.
- Map `Font.Weight` → JetBrains Mono PostScript name (`regular`, `medium/semibold` → Medium, `bold/heavy/black` → Bold; italic variants shipped).
- Provide both SwiftUI (`Font`) and UIKit (`UIFont`) resolvers. Both respect Dynamic Type via `relativeTo:` / `UIFontMetrics`.

```swift
enum AppFont {
    static func supports(_ language: AppLanguage) -> Bool { ... }
    static func resolve(style:language:weight:italic:) -> Font { ... }
    static func resolveUIFont(style:language:weight:italic:) -> UIFont { ... }
}
```

### `supports(_:)` rule

Only languages whose scripts the chosen font covers return `true`. For JetBrains Mono that means Latin + Cyrillic core. Kazakh Cyrillic extended characters (Қ, Ң, Ғ, Ү, Ұ, Ө, Һ, І) and all CJK glyphs are excluded by policy, even where the font technically covers some of them, to keep typography consistent within a language.

When adding a new app language, **explicitly** decide whether to extend `supports(_:)` after verifying glyph coverage.

---

## 5) Applying Fonts at Call Sites

### 5.1 SwiftUI — prefer `.appFont(...)`

```swift
Text(L("settings.title"))
    .appFont(.title2)

Text(amount)
    .appFont(.body, weight: .bold)

Text(version)
    .appFont(.custom(size: 12), weight: .bold)
```

The modifier subscribes to `LocalizationManager.shared.$currentLanguage`, so views re-render and swap fonts automatically when the user changes language.

### 5.2 SwiftUI — keep `.font(.system(...))` on `Image(systemName:)`

SF Symbols render at the font's cap height. Applying JetBrains Mono metrics shifts symbol sizes. Keep system font on symbols:

```swift
Image(systemName: "lightbulb")
    .font(.system(size: 64))  // not .appFont(.custom(size: 64))
```

For small symbol sizes tied to a text style (e.g., `.title2`), `.appFont(.title2)` is acceptable — the visual delta is imperceptible.

### 5.3 UIKit chrome — `AppFontAppearance`

`UINavigationBar` titles and `UITabBar` labels render through UIKit, not SwiftUI. Configure appearance proxies at launch and re-configure on language change:

```swift
init() {
    AppFontAppearance.shared.start()
}
```

`start()` subscribes to `LocalizationManager` and re-applies title attributes while **preserving** existing background/blur/shadow from the system or app-level overrides. Never call `configureWithDefaultBackground()` — that overwrites transparency and breaks scrolled-to-top behavior.

### 5.4 LaunchScreen — keep system font

`LaunchScreenView` uses the system font. SwiftUI-backed launch screens show after app init, so fonts are actually available, but keeping system font here avoids a FOUT-style visual glitch if the custom font cache is cold.

---

## 6) ShareExtension Exclusion

The extension must **not** bundle the TTFs:

- Keep TTFs only in the main app's `PBXFileSystemSynchronizedRootGroup`.
- Do **not** add the `Fonts/` directory to the extension target's synchronized group.
- Do **not** register `UIAppFonts` in the extension's `Info.plist`.
- Do **not** link `AppFont.swift` / `AppFontAppearance.swift` into the extension.

The extension keeps using the system font. This saves ~1.3 MB in extension size and avoids a second font registration on app launch.

---

## 7) Root-level Default

Apply the default font at the app root so unstyled `Text` inherits JetBrains Mono:

```swift
WindowGroup {
    ContentView()
        .appFont(.body)
}
```

SwiftUI's font is an environment value that cascades to any `Text` that does not explicitly set its own font.

---

## 8) Testing

`AppFontResolverTests` (Swift Testing) covers:

- `supports(_:)` returns `true` for supported languages, `false` for Kazakh/Chinese.
- `postScriptName(weight:italic:)` maps weights/italic correctly (regular → Regular, medium/semibold → Medium, bold/heavy/black → Bold, italic combinations).

Run via `./run_tests.sh`.

---

## 9) Checklist When Adding the Font to a New App

1. Copy `AppFont.swift` and `AppFontAppearance.swift` to `<App>/Shared/`.
2. Copy TTFs into `<App>/Fonts/`.
3. Add `UIAppFonts` array to the main app's `Info.plist`.
4. Verify Fonts directory is **not** in ShareExtension's synchronized group.
5. Call `AppFontAppearance.shared.start()` in app `init()`.
6. Add `.appFont(.body)` at the root.
7. Refactor existing `.font(.largeTitle)`, `.font(.title)`, `.font(.body)` etc. to `.appFont(...)` counterparts.
8. Leave `.font(.system(size:))` on SF Symbols >= ~32pt.
9. Update `AppFont.supports(_:)` for any new app languages — verify glyph coverage first.
10. Add `AppFontResolverTests` adapted to the app's `AppLanguage` cases.

---

## 10) Known Limitations

- Italic + medium renders as `JetBrainsMono-Italic` (no Medium-Italic shipped). Acceptable — italic is rarely paired with medium weight.
- Custom font cache is cold on first launch; UIKit chrome reads the registered font before the first render, so no FOUT in practice. If you observe one, delay `AppFontAppearance.shared.start()` until after `FirebaseApp.configure()` in the AppDelegate.
- JetBrains Mono is a monospace font. Numeric columns and tables align cleanly, but long-form Latin body text appears denser than system. Acceptable for a utility app — reconsider for content-heavy apps.
