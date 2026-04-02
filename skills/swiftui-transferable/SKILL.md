---
name: swiftui-transferable
description: SwiftUI Transferable protocol and drag-and-drop patterns including custom transfer representations, dropDestination, draggable, and clipboard integration. Use when implementing drag-and-drop or copy/paste.
---

# SwiftUI Transferable & Drag-and-Drop

Patterns for the Transferable protocol, drag-and-drop, clipboard integration, ShareLink, and file import/export in SwiftUI.

## When This Skill Activates

Use this skill when the user:
- Wants to implement drag-and-drop in SwiftUI
- Asks about the Transferable protocol or TransferRepresentation
- Needs copy/paste or clipboard integration
- Wants to use ShareLink with custom types
- Asks about .fileImporter or .fileExporter with Transferable
- Needs to reorder list items via drag
- Wants to offer an item in multiple transfer formats

## Decision Tree

```
What data transfer mechanism do you need?
|
+- SwiftUI-native drag/drop with modern types
|  +- Use Transferable protocol (iOS 16+)
|
+- Interop with UIKit or AppKit drag sessions
|  +- Use NSItemProvider (bridged via Transferable when possible)
|
+- Direct pasteboard read/write without SwiftUI modifiers
|  +- iOS: UIPasteboard.general
|  +- macOS: NSPasteboard.general
|
+- Sharing content via share sheet
|  +- Use ShareLink with Transferable items
|
+- File open/save panels
   +- Use .fileImporter / .fileExporter with Transferable
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `Transferable` protocol | iOS 16, macOS 13 | Core protocol |
| `CodableRepresentation` | iOS 16, macOS 13 | JSON-encoded Codable types |
| `DataRepresentation` | iOS 16, macOS 13 | Raw Data conversion |
| `FileRepresentation` | iOS 16, macOS 13 | File-based transfers |
| `ProxyRepresentation` | iOS 16, macOS 13 | Delegate to another Transferable |
| `.draggable()` | iOS 16, macOS 13 | Drag source modifier |
| `.dropDestination(for:action:isTargeted:)` | iOS 16, macOS 13 | Drop target modifier |
| `ShareLink` | iOS 16, macOS 13 | Share sheet integration |
| `.fileImporter()` | iOS 14, macOS 11 | File open panel |
| `.fileExporter()` | iOS 14, macOS 11 | File save panel |
| `PasteButton` | iOS 16, macOS 13 | System paste button |

## Transferable Protocol & Representations

### CodableRepresentation

Best for Codable model types shared within your app or with other apps that understand your content type:

```swift
struct TodoItem: Codable, Transferable {
    var title: String
    var isCompleted: Bool

    static var transferRepresentation: some TransferRepresentation {
        CodableRepresentation(contentType: .json)
    }
}
```

### DataRepresentation

For custom binary encoding or when you need manual control over serialization:

```swift
struct Diagram: Transferable {
    var paths: [Path]

    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(contentType: .diagram) { diagram in
            try DiagramEncoder().encode(diagram)
        } importing: { data in
            try DiagramDecoder().decode(from: data)
        }
    }
}
```

### FileRepresentation

For large payloads best transferred as files:

```swift
struct VideoProject: Transferable {
    var fileURL: URL

    static var transferRepresentation: some TransferRepresentation {
        FileRepresentation(contentType: .mpeg4Movie) { project in
            SentTransferredFile(project.fileURL)
        } importing: { received in
            let destination = FileManager.default.temporaryDirectory
                .appendingPathComponent(received.file.lastPathComponent)
            try FileManager.default.copyItem(at: received.file, to: destination)
            return Self(fileURL: destination)
        }
    }
}
```

### ProxyRepresentation

Delegate transfer to an existing Transferable type (String, Image, URL, etc.):

```swift
struct Contact: Transferable {
    var name: String
    var email: String

    static var transferRepresentation: some TransferRepresentation {
        ProxyRepresentation(exporting: \.name)
    }
}
```

## Multi-Representation

Offer the same item in multiple formats so the receiver picks the best one. Representations are tried in declaration order:

```swift
struct RichNote: Codable, Transferable {
    var title: String
    var body: String

    static var transferRepresentation: some TransferRepresentation {
        // Preferred: full fidelity
        CodableRepresentation(contentType: .json)
        // Fallback: plain text for any text receiver
        ProxyRepresentation(exporting: \.body)
    }
}
```

## .draggable() Modifier

```swift
LazyVGrid(columns: [GridItem(.adaptive(minimum: 100))]) {
    ForEach(photos) { photo in
        PhotoThumbnail(photo: photo)
            .draggable(photo) {
                // Custom drag preview
                PhotoThumbnail(photo: photo)
                    .frame(width: 80, height: 80)
                    .clipShape(RoundedRectangle(cornerRadius: 8))
            }
    }
}
```

## .dropDestination() Modifier

```swift
@State private var droppedPhotos: [Photo] = []
@State private var isTargeted = false

ZStack {
    RoundedRectangle(cornerRadius: 12)
        .fill(isTargeted ? Color.accentColor.opacity(0.2) : Color.gray.opacity(0.1))
    Text(droppedPhotos.isEmpty ? "Drop photos here" : "\(droppedPhotos.count) photos")
}
.dropDestination(for: Photo.self) { photos, location in
    droppedPhotos.append(contentsOf: photos)
    return true
} isTargeted: { targeted in
    isTargeted = targeted
}
```

## Reordering Lists with Transferable

Use `.onMove` for simple list reordering, or combine `.draggable` with `.dropDestination` for custom reorder logic:

```swift
// Simple reordering
List {
    ForEach(items) { item in
        Text(item.title)
    }
    .onMove { source, destination in
        items.move(fromOffsets: source, toOffset: destination)
    }
}
```

Custom drag-based reordering:

```swift
ForEach(items) { item in
    Text(item.title)
        .draggable(item)
        .dropDestination(for: TodoItem.self) { droppedItems, _ in
            guard let source = droppedItems.first,
                  let fromIndex = items.firstIndex(of: source),
                  let toIndex = items.firstIndex(of: item) else { return false }
            items.move(fromOffsets: IndexSet(integer: fromIndex),
                       toOffset: toIndex > fromIndex ? toIndex + 1 : toIndex)
            return true
        }
}
```

## ShareLink with Transferable

```swift
// Single item
ShareLink(item: note, preview: SharePreview(note.title))

// Collection
ShareLink(items: selectedPhotos) { photo in
    SharePreview(photo.caption, image: photo.thumbnail)
}
```

## Clipboard Integration

### PasteButton (Preferred)

The system `PasteButton` grants paste permission without a confirmation prompt:

```swift
PasteButton(payloadType: String.self) { strings in
    guard let text = strings.first else { return }
    content = text
}
```

### Direct Pasteboard Access

```swift
// iOS
UIPasteboard.general.string = note.body
if let text = UIPasteboard.general.string { content = text }

// macOS
NSPasteboard.general.clearContents()
NSPasteboard.general.setString(note.body, forType: .string)
```

## File Importer & Exporter

```swift
// Import
@State private var showImporter = false

Button("Import") { showImporter = true }
    .fileImporter(
        isPresented: $showImporter,
        allowedContentTypes: [.plainText, .json]
    ) { result in
        if case .success(let url) = result {
            guard url.startAccessingSecurityScopedResource() else { return }
            defer { url.stopAccessingSecurityScopedResource() }
            document = try? TextDocument(fileURL: url)
        }
    }

// Export
@State private var showExporter = false

Button("Export") { showExporter = true }
    .fileExporter(
        isPresented: $showExporter,
        document: document,
        contentType: .plainText,
        defaultFilename: "export.txt"
    ) { result in
        if case .failure(let error) = result {
            print("Export failed: \(error.localizedDescription)")
        }
    }
```

## Good and Bad Patterns

### Transferable Conformance

```swift
// ❌ Using .data — too generic, receivers can't identify your content
CodableRepresentation(contentType: .data)

// ✅ Use a specific UTType
CodableRepresentation(contentType: .json)
```

### Drop Destination

```swift
// ❌ Ignoring the return value — system uses it for the drop animation
.dropDestination(for: Photo.self) { photos, location in
    droppedPhotos.append(contentsOf: photos)
    // Missing return — defaults to Void which won't compile
}

// ✅ Return true on success, false on failure
.dropDestination(for: Photo.self) { photos, location in
    droppedPhotos.append(contentsOf: photos)
    return true
} isTargeted: { targeted in
    isTargeted = targeted
}
```

### Security-Scoped Resources

```swift
// ❌ Reading imported file without security scope — may fail silently
if case .success(let url) = result {
    let data = try? Data(contentsOf: url)
}

// ✅ Always bracket with security-scoped access
if case .success(let url) = result {
    guard url.startAccessingSecurityScopedResource() else { return }
    defer { url.stopAccessingSecurityScopedResource() }
    let data = try? Data(contentsOf: url)
}
```

### Multi-Representation Ordering

```swift
// ❌ Lossy representation first — receivers prefer it over full-fidelity
static var transferRepresentation: some TransferRepresentation {
    ProxyRepresentation(exporting: \.title)
    CodableRepresentation(contentType: .json)
}

// ✅ Highest fidelity first, fallbacks after
static var transferRepresentation: some TransferRepresentation {
    CodableRepresentation(contentType: .json)
    ProxyRepresentation(exporting: \.title)
}
```

## References

- [Transferable](https://developer.apple.com/documentation/coretransferable/transferable)
- [TransferRepresentation](https://developer.apple.com/documentation/coretransferable/transferrepresentation)
- [draggable(_:preview:)](https://developer.apple.com/documentation/swiftui/view/draggable(_:preview:))
- [dropDestination(for:action:isTargeted:)](https://developer.apple.com/documentation/swiftui/view/dropdestination(for:action:istargeted:))
- [ShareLink](https://developer.apple.com/documentation/swiftui/sharelink)
- [FileDocument](https://developer.apple.com/documentation/swiftui/filedocument)
