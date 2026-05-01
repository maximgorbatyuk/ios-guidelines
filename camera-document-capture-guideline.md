# Camera Document Capture Guideline for iOS Apps

This guideline is based on the implementation in:

- `EVChargingTracker` (canonical, fully reviewed)
- `journey-wallet` (original reference, has known issues)
- `CreativityHub` (existing implementation, needs alignment)

It defines a reusable standard for adding camera-based photo capture to document flows in any iOS app in this workspace.

---

## 1) Purpose

Allow users to capture photos directly from the camera when adding documents, as an alternative to file import and photo library selection. The captured photo is saved as a JPEG document and the user is immediately offered a chance to rename it before continuing.

---

## 2) Cross-App Comparison Summary

| Area | Journey Wallet | CreativityHub | EVChargingTracker | Recommended Standard |
|---|---|---|---|---|
| Source picker UI | Custom `List` in `DocumentPickerView` sheet | Custom `List` in `DocumentPickerView` sheet | Dedicated `DocumentSourcePickerView` sheet with `.presentationDetents([.medium])` | Dedicated source picker view with half-height detent |
| Camera wrapper | `CameraView` inside `DocumentPickerView.swift` | `CameraView` inside `DocumentPickerView.swift` | `CameraView` in `Shared/CameraView.swift` | Separate reusable file in `Shared/` |
| Camera availability guard | None | None | `UIImagePickerController.isSourceTypeAvailable(.camera)` | Always guard; hide button when unavailable |
| Dismissal pattern | `picker.dismiss(animated: true)` (manual) | `picker.dismiss(animated: true)` (manual) | SwiftUI-managed via binding (`showingCamera = false`) | Let SwiftUI manage dismissal; never call `picker.dismiss` manually |
| Filename strategy | `UUID().uuidString` | `UUID().uuidString` | `UUID().uuidString` | Always use UUID, never timestamp |
| Post-capture edit | Name entry sheet (pending document queue) | Name entry sheet (pending document queue) | Auto-open `DocumentEditView` via `onDismiss` chaining | Auto-open edit view after camera dismisses |
| Insert failure handling | No guard | No guard | `guard let rowId` — returns `nil` on failure | Always guard insert result; do not open edit for failed saves |
| Analytics source tracking | No `source` property | No `source` property | `source: "camera"` / `"photo_library"` in `document_added` event | Include `source` in analytics to distinguish import methods |
| Navigation container | `NavigationView` (deprecated) | `NavigationView` (deprecated) | `NavigationStack` | Always use `NavigationStack` |
| `NSCameraUsageDescription` | Not set | Not set | Set in `Info.plist` + localized `InfoPlist.strings` | Required for App Store; must be localized |
| Source enum placement | No enum (inline buttons) | No enum (inline buttons) | `DocumentSource` enum in own file | Dedicated enum file in Documents directory |

---

## 3) Canonical Architecture

### File structure

```
{AppName}/
├── Documents/
│   ├── DocumentSource.swift              # enum DocumentSource { files, photos, camera }
│   ├── DocumentSourcePickerView.swift    # SwiftUI sheet with styled rows
│   ├── DocumentsListView.swift           # Main list, chains pickers via onDismiss
│   ├── DocumentsViewModel.swift          # importImageData returns CarDocument?
│   ├── DocumentEditView.swift            # Rename sheet (existing)
│   └── DocumentPreviewView.swift         # Preview (existing)
├── Shared/
│   └── CameraView.swift                  # Reusable UIViewControllerRepresentable
```

### Data flow

```
User taps +
  → DocumentSourcePickerView (half-height sheet)
  → User selects source
  → Sheet dismisses
  → onDismiss chains the appropriate picker:

  [Files]  → .fileImporter → importFile()
  [Photos] → .photosPicker → importImageData(source: "photo_library")
  [Camera] → .fullScreenCover(CameraView)
               → UIImage captured
               → JPEG compression (0.8 quality)
               → importImageData(source: "camera") → returns CarDocument?
               → Camera dismisses
               → onDismiss opens DocumentEditView for the new document
```

---

## 4) Implementation Details

### 4.1 DocumentSource enum

Separate file in the `Documents/` directory:

```swift
enum DocumentSource {
    case files
    case photos
    case camera
}
```

### 4.2 CameraView (Shared/CameraView.swift)

Reusable `UIViewControllerRepresentable` wrapping `UIImagePickerController`:

```swift
import SwiftUI
import UIKit

struct CameraView: UIViewControllerRepresentable {
    let onCapture: (UIImage?) -> Void

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(onCapture: onCapture)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let onCapture: (UIImage?) -> Void

        init(onCapture: @escaping (UIImage?) -> Void) {
            self.onCapture = onCapture
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            let image = info[.originalImage] as? UIImage
            onCapture(image)
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            onCapture(nil)
        }
    }
}
```

**Critical rules:**
- Never call `picker.dismiss(animated: true)` in delegate methods. SwiftUI manages the `fullScreenCover` lifecycle via the `isPresented` binding.
- Always import `UIKit` explicitly (project convention for `UIViewControllerRepresentable` files).

### 4.3 DocumentSourcePickerView

Custom SwiftUI sheet replacing `confirmationDialog`:

```swift
struct DocumentSourcePickerView: SwiftUI.View {
    let onSelect: (DocumentSource) -> Void
    @Environment(\.dismiss) private var dismiss

    var body: some SwiftUI.View {
        NavigationStack {
            List {
                Section {
                    // Files row: icon "folder.fill" (.blue)
                    // Photos row: icon "photo.fill" (.green)
                    // Camera row: icon "camera.fill" (.orange) — conditionally shown
                    if UIImagePickerController.isSourceTypeAvailable(.camera) {
                        // camera row
                    }
                } footer: {
                    Text(L("document.source.supported_formats"))
                }
            }
            .navigationTitle(L("document.source.title"))
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button(L("Cancel")) { dismiss() }
                }
            }
        }
        .presentationDetents([.medium])
    }
}
```

**Each row has:**
- Colored SF Symbol icon (24pt frame)
- Title (`L()` localized)
- Description caption (`L()` localized, `.secondary` color)

**Critical rules:**
- Use `NavigationStack`, never `NavigationView` (deprecated).
- Use `.presentationDetents([.medium])` for half-height sheet.
- Guard camera row with `UIImagePickerController.isSourceTypeAvailable(.camera)`.

### 4.4 Sheet-to-picker chaining pattern

The source picker sheet must fully dismiss before presenting the next picker (file importer, photo picker, or camera). This avoids nested presentation conflicts.

```swift
// State
@State private var showingSourcePicker = false
@State private var selectedSource: DocumentSource?
@State private var showingImportPicker = false
@State private var showingPhotoPicker = false
@State private var showingCamera = false

// Sheet with onDismiss chaining
.sheet(isPresented: $showingSourcePicker, onDismiss: {
    handleSourceSelection()
}) {
    DocumentSourcePickerView { source in
        selectedSource = source
        showingSourcePicker = false
    }
}

// Chaining handler
private func handleSourceSelection() {
    guard let source = selectedSource else { return }
    selectedSource = nil
    switch source {
    case .files:  showingImportPicker = true
    case .photos: showingPhotoPicker = true
    case .camera: showingCamera = true
    }
}
```

### 4.5 Camera-to-edit chaining pattern

After the camera captures a photo and saves it, the edit view should open automatically so the user can rename the document. This also uses `onDismiss` to avoid presentation conflicts.

```swift
// State
@State private var capturedDocument: CarDocument?  // or Document, depending on app model

// fullScreenCover with onDismiss chaining
.fullScreenCover(isPresented: $showingCamera, onDismiss: {
    if let doc = capturedDocument {
        capturedDocument = nil
        documentToEdit = doc
    }
}) {
    CameraView { image in
        showingCamera = false       // triggers SwiftUI dismissal
        handleCapturedImage(image)  // saves document, stores in capturedDocument
    }
}

// Capture handler
private func handleCapturedImage(_ image: UIImage?) {
    guard let image = image,
          let data = image.jpegData(compressionQuality: 0.8) else {
        return
    }
    let fileName = "capture_\(UUID().uuidString).jpg"
    capturedDocument = viewModel.importImageData(
        data, fileName: fileName, customTitle: nil, source: "camera"
    )
}
```

### 4.6 ViewModel: importImageData return value

The method must return the persisted document (with its database ID) so the view can open the edit screen. It must guard against insert failure.

```swift
@discardableResult
func importImageData(_ data: Data, fileName: String, customTitle: String?, source: String = "photo_library") -> Document? {
    // ... save file, create document object ...

    guard let rowId = documentsRepo?.insertRecord(doc) else {
        return nil  // do NOT return a document with nil ID
    }
    doc.id = rowId

    analytics.trackEvent("document_added", properties: [
        "screen": "documents_list",
        "file_type": fileType,
        "source": source     // "camera" or "photo_library"
    ])
    loadData()
    return doc
}
```

**Critical rules:**
- `@discardableResult` so photo library callers can ignore the return value.
- `guard let rowId` — never return a document that failed to persist.
- Include `source` in analytics event properties.

---

## 5) NSCameraUsageDescription

### Info.plist

Add the base English string:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is needed to take photos of your documents</string>
```

### InfoPlist.strings (localized)

Create `InfoPlist.strings` in each `.lproj` directory with the localized camera description:

```
// en.lproj/InfoPlist.strings
"NSCameraUsageDescription" = "Camera access is needed to take photos of your documents";
```

Translate for all supported languages. This string appears in the iOS system permission dialog — users see it in their device language.

---

## 6) Localization Keys

### Required keys per language

| Key | Purpose | Example (EN) |
|-----|---------|--------------|
| `"Take Photo"` | Camera button title | Take Photo |
| `"document.source.title"` | Source picker navigation title | Add Document |
| `"document.source.files.description"` | Files row description | Import PDF, JPEG, PNG, or HEIC files |
| `"document.source.photos.description"` | Photos row description | Choose from your photo library |
| `"document.source.camera.description"` | Camera row description | Take a photo with your camera |
| `"document.source.supported_formats"` | Section footer | Supported formats: PDF, JPEG, PNG, HEIC |

### Language matrix

| Key | EN | DE | RU | KK | TR | UK |
|-----|----|----|----|----|----|----|
| `Take Photo` | Take Photo | Foto aufnehmen | Сделать фото | Фото түсіру | Fotoğraf çek | Зробити фото |
| `document.source.title` | Add Document | Dokument hinzufügen | Добавить документ | Құжат қосу | Belge Ekle | Додати документ |
| `document.source.files.description` | Import PDF, JPEG, PNG, or HEIC files | PDF-, JPEG-, PNG- oder HEIC-Dateien importieren | Импорт файлов PDF, JPEG, PNG или HEIC | PDF, JPEG, PNG немесе HEIC файлдарын импорттау | PDF, JPEG, PNG veya HEIC dosyalarını içe aktar | Імпорт файлів PDF, JPEG, PNG або HEIC |
| `document.source.photos.description` | Choose from your photo library | Aus der Fotobibliothek auswählen | Выбрать из медиатеки | Фото кітапханадан таңдау | Fotoğraf kitaplığından seç | Вибрати з медіатеки |
| `document.source.camera.description` | Take a photo with your camera | Ein Foto mit der Kamera aufnehmen | Сделать фото камерой | Камерамен фото түсіру | Kamerayla fotoğraf çek | Зробити фото камерою |
| `document.source.supported_formats` | Supported formats: PDF, JPEG, PNG, HEIC | Unterstützte Formate: PDF, JPEG, PNG, HEIC | Поддерживаемые форматы: PDF, JPEG, PNG, HEIC | Қолдау көрсетілетін форматтар: PDF, JPEG, PNG, HEIC | Desteklenen formatlar: PDF, JPEG, PNG, HEIC | Підтримувані формати: PDF, JPEG, PNG, HEIC |

Note: CreativityHub supports only EN, RU, KK. Journey Wallet and EVChargingTracker support all six languages.

---

## 7) JPEG Compression

- Quality: `0.8` (80%) — reasonable balance of quality vs. file size.
- Applied immediately after capture via `image.jpegData(compressionQuality: 0.8)`.
- Filename format: `capture_{UUID}.jpg` — always use `UUID().uuidString`, never timestamp (avoids collisions).
- No additional image processing (resize, crop, metadata strip) is applied.

---

## 8) Common Mistakes to Avoid

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| Calling `picker.dismiss(animated: true)` in CameraView delegate | Conflicts with SwiftUI's `fullScreenCover` dismissal management, can cause stuck state or re-presentation failures | Only set the binding to `false` in the `onCapture` callback; let SwiftUI handle dismissal |
| Using `confirmationDialog` for source picker | Plain text buttons with no icons or descriptions; poor UX | Use a dedicated SwiftUI sheet with styled `List` rows |
| No camera availability check | Crashes on Simulator and devices without camera | Guard with `UIImagePickerController.isSourceTypeAvailable(.camera)` |
| Using `NavigationView` | Deprecated; causes split-view issues on iPad in sheets | Use `NavigationStack` |
| Timestamp-based filenames (`Date().timeIntervalSince1970`) | Second-level precision; collision risk with rapid captures | Use `UUID().uuidString` |
| Returning document with nil ID after failed insert | Edit view opens for a document that doesn't exist in the database; rename silently fails | `guard let rowId` — return `nil` when insert fails |
| Presenting edit sheet while camera is still visible | Nested presentation conflict; one presentation is ignored | Use `onDismiss` on `fullScreenCover` to chain the edit sheet after camera fully dismisses |
| No `NSCameraUsageDescription` | App rejected by App Store review | Add to `Info.plist` and localize via `InfoPlist.strings` |
| Hardcoded English camera permission string | Non-English users see English in system dialog | Create `InfoPlist.strings` in every `.lproj` directory |
| No `source` in analytics | Cannot distinguish camera captures from photo library imports | Include `"source": "camera"` or `"photo_library"` in the `document_added` event |

---

## 9) Checklist for New App Implementation

- [ ] Create `Shared/CameraView.swift` — no manual `picker.dismiss`, explicit `import UIKit`
- [ ] Create `Documents/DocumentSource.swift` — enum with `files`, `photos`, `camera` cases
- [ ] Create `Documents/DocumentSourcePickerView.swift` — `NavigationStack`, `.presentationDetents([.medium])`, camera row guarded by `isSourceTypeAvailable`
- [ ] Update document list view — replace `confirmationDialog` with `.sheet` + `onDismiss` chaining
- [ ] Add camera `.fullScreenCover` with `onDismiss` → auto-open edit view for captured document
- [ ] Update ViewModel `importImageData` — return `CarDocument?`, `@discardableResult`, guard insert failure, include `source` in analytics
- [ ] Add `NSCameraUsageDescription` to `Info.plist`
- [ ] Create `InfoPlist.strings` in all `.lproj` directories with localized camera permission
- [ ] Add localization keys to all `Localizable.strings` files (see section 6)
- [ ] Build and verify on Simulator (camera button should be hidden)
- [ ] Test on physical device (camera capture → JPEG save → edit view opens)

---

## 10) Alignment Tasks for Existing Apps

### Journey Wallet

- [ ] Extract `CameraView` from `DocumentPickerView.swift` to `Shared/CameraView.swift`
- [ ] Remove `picker.dismiss(animated: true)` from CameraView Coordinator
- [ ] Add `UIImagePickerController.isSourceTypeAvailable(.camera)` guard
- [ ] Replace `NavigationView` with `NavigationStack` in `DocumentPickerView`
- [ ] Add `source` property to analytics events
- [ ] Add `NSCameraUsageDescription` to `Info.plist`
- [ ] Create localized `InfoPlist.strings` in all 6 `.lproj` directories
- [ ] Guard insert failure in document save flow (return nil instead of partial document)

### CreativityHub

- [ ] Extract `CameraView` from `DocumentPickerView.swift` to `Shared/CameraView.swift`
- [ ] Remove `picker.dismiss(animated: true)` from CameraView Coordinator
- [ ] Add `UIImagePickerController.isSourceTypeAvailable(.camera)` guard
- [ ] Replace `NavigationView` with `NavigationStack` in `DocumentPickerView`
- [ ] Add `source` property to analytics events
- [ ] Add `NSCameraUsageDescription` to `Info.plist`
- [ ] Create localized `InfoPlist.strings` in EN, RU, KK `.lproj` directories
- [ ] Guard insert failure in document save flow
- [ ] Consider replacing batch pending-document queue with auto-open edit view pattern (simpler UX for single captures)
