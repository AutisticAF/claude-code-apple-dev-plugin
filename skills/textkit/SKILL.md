---
name: textkit
description: TextKit 2 patterns for text layout, editing, and rendering including NSTextLayoutManager, NSTextContentStorage, custom rendering, and practical workarounds from STTextView. Use when building text editors, custom text views, or working with TextKit 2 APIs.
---

> **First step:** Tell the user: "textkit skill loaded."

# TextKit 2

TextKit 2 is Apple's modern text layout and rendering framework, introduced at WWDC 2021 as a replacement for TextKit 1 (the NSLayoutManager-based stack). It provides viewport-based layout, non-contiguous text support, and a more composable architecture. However, it has significant rough edges — STTextView (a production TextKit 2 text editor) has filed 20+ bug reports with Apple.

This skill covers both the API and the practical reality of working with TextKit 2, including known limitations and workarounds.

## When This Skill Activates

- User is building a custom text view or text editor
- User asks about NSTextLayoutManager, NSTextContentStorage, or TextKit 2
- User needs custom text rendering (line numbers, annotations, highlights)
- User is migrating from TextKit 1 (NSLayoutManager) to TextKit 2
- User encounters TextKit 2 bugs or unexpected behavior
- User asks about STTextView or custom text editing components
- User needs viewport-based text layout for large documents
- User wants to customize text fragment rendering

## Decision Tree

```
What text feature do you need?
|
+-- Display styled text (no editing)
|   +-- Simple? Use SwiftUI Text or NSAttributedString.draw(in:)
|   +-- Glyph-level control? Use CoreText (CTFrame/CTLine)
|
+-- Editable text field
|   +-- Standard text input? Use TextField / TextEditor
|   +-- Rich text editing? Use NSTextView / UITextView
|
+-- Custom text editor (code editor, annotations, etc.)
|   +-- Use TextKit 2 directly (NSTextLayoutManager stack)
|   +-- Or use STTextView as a foundation
|
+-- Need TextKit 1 compatibility
    +-- Use NSLayoutManager (still supported, not deprecated)
```

## API Availability

| API | iOS | macOS | Notes |
|-----|-----|-------|-------|
| `NSTextLayoutManager` | 15.0+ | 12.0+ | Core TextKit 2 layout engine |
| `NSTextContentManager` | 15.0+ | 12.0+ | Abstract content protocol |
| `NSTextContentStorage` | 15.0+ | 12.0+ | Concrete backing store |
| `NSTextLayoutFragment` | 15.0+ | 12.0+ | Rendered text fragment |
| `NSTextElement` | 15.0+ | 12.0+ | Abstract text unit |
| `NSTextParagraph` | 15.0+ | 12.0+ | Paragraph-level element |
| `NSTextViewportLayoutController` | 15.0+ | 12.0+ | Viewport-based layout |
| `NSTextRange` | 15.0+ | 12.0+ | Range in text content |
| `NSTextLocation` | 15.0+ | 12.0+ | Position in text content |
| `NSTextSelectionNavigation` | 15.0+ | 12.0+ | Selection management |

## Architecture Overview

TextKit 2 separates concerns into distinct layers:

```
NSTextContentStorage (model: stores attributed string)
        |
NSTextLayoutManager (layout: arranges text into fragments)
        |
NSTextLayoutFragment (rendering: visual representation of a paragraph)
        |
NSTextViewportLayoutController (viewport: manages visible region)
        |
View (NSTextView / UITextView / custom view)
```

### Key Difference from TextKit 1

TextKit 1 (`NSLayoutManager`) performs contiguous layout — it lays out all text from the beginning. TextKit 2 uses **viewport-based layout** — only visible text is laid out, enabling efficient rendering of very large documents.

## Setting Up the TextKit 2 Stack

### Manual Stack (Without NSTextView)

```swift
import UIKit

// 1. Content storage (model)
let textContentStorage = NSTextContentStorage()
textContentStorage.attributedString = NSAttributedString(
    string: "Hello, TextKit 2!",
    attributes: [.font: UIFont.systemFont(ofSize: 16)]
)

// 2. Layout manager
let textLayoutManager = NSTextLayoutManager()
textLayoutManager.textContainer = NSTextContainer(size: CGSize(width: 300, height: 0))
textLayoutManager.textContainer?.lineFragmentPadding = 5

// 3. Connect them
textContentStorage.addTextLayoutManager(textLayoutManager)
```

### Using with NSTextView (macOS)

```swift
import AppKit

let textView = NSTextView(usingTextLayoutManager: true)
// Access the TextKit 2 stack:
let layoutManager = textView.textLayoutManager!
let contentStorage = layoutManager.textContentManager as! NSTextContentStorage
```

### Using with UITextView (iOS)

```swift
import UIKit

// UITextView uses TextKit 2 by default on iOS 16+
let textView = UITextView()
let layoutManager = textView.textLayoutManager!
```

## NSTextContentStorage

The model layer storing the text as `NSAttributedString`.

### Reading and Writing Text

```swift
let contentStorage = NSTextContentStorage()

// Set text
contentStorage.attributedString = NSAttributedString(string: "Hello")

// Read text
let fullText = contentStorage.attributedString?.string ?? ""

// Replace a range
let documentRange = contentStorage.documentRange
if let start = contentStorage.location(documentRange.location, offsetBy: 5),
   let end = contentStorage.location(start, offsetBy: 3) {
    let range = NSTextRange(location: start, end: end)
    contentStorage.replaceContents(
        in: range,
        with: [NSAttributedString(string: "replacement")]
    )
}
```

### Known Issue: replaceContents and Multi-Cursor

STTextView discovered that `NSTextLayoutManager._fixSelectionAfterChangeInCharacterRange` unexpectedly resets text selections after `replaceContents`, breaking multi-cursor editing (FB9925647). STTextView works around this by subclassing `NSTextContentStorage` and manually deduplicating/restoring selections after replacement.

## NSTextLayoutManager

The layout engine that arranges text into fragments.

### Enumerating Layout Fragments

```swift
let layoutManager: NSTextLayoutManager = // ...

// Enumerate all visible fragments
layoutManager.enumerateTextLayoutFragments(
    from: layoutManager.documentRange.location,
    options: [.ensuresLayout, .ensuresExtraLineFragment]
) { fragment in
    let frame = fragment.layoutFragmentFrame
    let paragraphRange = fragment.rangeInElement
    print("Fragment at \(frame), range: \(paragraphRange)")
    return true // continue enumeration
}
```

### Getting Fragment for a Location

```swift
if let fragment = layoutManager.textLayoutFragment(for: location) {
    let frame = fragment.layoutFragmentFrame
    // Use frame for positioning UI elements (annotations, line numbers, etc.)
}
```

### Text Selections

```swift
// Get current selections
let selections = layoutManager.textSelections

// Set selections programmatically
let selection = NSTextSelection(range: range, affinity: .downstream, granularity: .character)
layoutManager.textSelections = [selection]
```

### Known Issue: Selection Notifications

TextKit 2 does not provide consistent cross-platform selection change notifications. STTextView works around this by subclassing `NSTextLayoutManager` and posting a custom notification in the `textSelections` didSet observer.

## NSTextLayoutFragment

Each fragment represents the visual rendering of a text paragraph.

### Custom Fragment Rendering

Subclass `NSTextLayoutFragment` for custom drawing (syntax highlighting overlays, annotations, etc.):

```swift
class CustomTextLayoutFragment: NSTextLayoutFragment {
    override func draw(at point: CGPoint, in context: CGContext) {
        // Draw the standard text first
        super.draw(at: point, in: context)

        // Add custom overlays
        context.saveGState()
        context.setFillColor(NSColor.yellow.withAlphaComponent(0.3).cgColor)
        context.fill(CGRect(x: point.x, y: point.y, width: renderingSurfaceBounds.width, height: renderingSurfaceBounds.height))
        context.restoreGState()
    }
}
```

Register the custom fragment via `NSTextLayoutManagerDelegate`:

```swift
func textLayoutManager(
    _ textLayoutManager: NSTextLayoutManager,
    textLayoutFragmentFor location: NSTextLocation,
    in textElement: NSTextElement
) -> NSTextLayoutFragment {
    CustomTextLayoutFragment(textElement: textElement, range: textElement.elementRange)
}
```

### Known Issue: Extra Line Fragments

TextKit 2 may create unexpected additional line fragments at the end of documents or after certain attribute changes. STTextView filed multiple reports about this (FB9856587, FB10901256). Workaround: filter out zero-height or anomalous fragments when enumerating.

## NSTextViewportLayoutController

Manages which text is laid out based on the visible viewport.

### Viewport Layout Delegate

```swift
class MyViewportDelegate: NSObject, NSTextViewportLayoutControllerDelegate {
    func viewportBounds(for textViewportLayoutController: NSTextViewportLayoutController) -> CGRect {
        // Return the visible rect of your scroll view
        return scrollView.documentVisibleRect
    }

    func textViewportLayoutControllerWillLayout(
        _ textViewportLayoutController: NSTextViewportLayoutController
    ) {
        // Prepare for layout pass (e.g., remove stale fragment views)
    }

    func textViewportLayoutController(
        _ textViewportLayoutController: NSTextViewportLayoutController,
        configureRenderingSurfaceFor textLayoutFragment: NSTextLayoutFragment
    ) {
        // Add/update the fragment's view in your text container view
        let fragmentView = fragmentViewMap[textLayoutFragment] ?? createFragmentView(for: textLayoutFragment)
        fragmentView.frame = textLayoutFragment.layoutFragmentFrame
    }

    func textViewportLayoutControllerDidLayout(
        _ textViewportLayoutController: NSTextViewportLayoutController
    ) {
        // Finalize layout (update scroll content size, etc.)
    }
}
```

## Building a Custom Text View

A minimal custom text view using TextKit 2 directly (the approach STTextView takes):

```swift
import AppKit

class MinimalTextView: NSView {
    let textContentStorage = NSTextContentStorage()
    let textLayoutManager = NSTextLayoutManager()
    private let textContainer = NSTextContainer()

    override init(frame: NSRect) {
        super.init(frame: frame)

        textContainer.size = NSSize(width: frame.width, height: 0)
        textContainer.lineFragmentPadding = 5
        textLayoutManager.textContainer = textContainer
        textContentStorage.addTextLayoutManager(textLayoutManager)

        textContentStorage.attributedString = NSAttributedString(
            string: "Hello, custom TextKit 2 view!",
            attributes: [
                .font: NSFont.monospacedSystemFont(ofSize: 14, weight: .regular),
                .foregroundColor: NSColor.textColor
            ]
        )
    }

    required init?(coder: NSCoder) { fatalError() }

    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)
        guard let context = NSGraphicsContext.current?.cgContext else { return }

        textLayoutManager.enumerateTextLayoutFragments(
            from: textLayoutManager.documentRange.location,
            options: [.ensuresLayout]
        ) { fragment in
            fragment.draw(at: fragment.layoutFragmentFrame.origin, in: context)
            return true
        }
    }
}
```

## Line Numbers (Gutter)

A common text editor feature. Use layout fragment enumeration to align line numbers:

```swift
func drawLineNumbers(in context: CGContext, gutterWidth: CGFloat) {
    var lineNumber = 1

    textLayoutManager.enumerateTextLayoutFragments(
        from: textLayoutManager.documentRange.location,
        options: [.ensuresLayout]
    ) { fragment in
        let frame = fragment.layoutFragmentFrame
        let lineNumberString = "\(lineNumber)" as NSString

        let attributes: [NSAttributedString.Key: Any] = [
            .font: NSFont.monospacedDigitSystemFont(ofSize: 12, weight: .regular),
            .foregroundColor: NSColor.secondaryLabelColor
        ]

        let size = lineNumberString.size(withAttributes: attributes)
        let origin = CGPoint(
            x: gutterWidth - size.width - 4,
            y: frame.origin.y + (frame.height - size.height) / 2
        )
        lineNumberString.draw(at: origin, withAttributes: attributes)

        lineNumber += 1
        return true
    }
}
```

## Hit Testing and Caret Placement

```swift
// Point → text location
func textLocation(for point: CGPoint) -> NSTextLocation? {
    // Adjust point to text container coordinates
    let adjustedPoint = CGPoint(x: point.x - gutterWidth, y: point.y)

    // Find the nearest text location
    let fraction = UnsafeMutablePointer<CGFloat>.allocate(capacity: 1)
    defer { fraction.deallocate() }

    guard let location = textLayoutManager.location(
        interactingAt: adjustedPoint,
        inContainerAt: textLayoutManager.documentRange.location
    ) else { return nil }

    return location
}

// Text location → point (for drawing caret)
func caretRect(for location: NSTextLocation) -> CGRect? {
    guard let fragment = textLayoutManager.textLayoutFragment(for: location) else {
        return nil
    }

    // Get the line fragment for precise positioning
    let selectionNav = textLayoutManager.textSelectionNavigation
    // Use selection geometry for precise caret rectangles
    let selectionRects = textLayoutManager.enumerateRenderingAttributes(
        from: location, reverse: false
    ) { _, _ in return true }

    return nil // simplified — real implementation uses textSelectionSegmentFrame
}
```

## Migration from TextKit 1

| TextKit 1 | TextKit 2 | Notes |
|-----------|-----------|-------|
| `NSLayoutManager` | `NSTextLayoutManager` | Different delegate methods |
| `NSTextStorage` | `NSTextContentStorage` | Wraps `NSAttributedString` |
| `NSTextContainer` | `NSTextContainer` | Same class, shared |
| `layoutManager(_:lineFragmentRectFor:...)` | `NSTextLayoutFragment` subclass | Custom layout |
| `NSLayoutManager.enumerateLineFragments` | `enumerateTextLayoutFragments` | Viewport-aware |
| `NSRange` | `NSTextRange` / `NSTextLocation` | Opaque locations, not integers |
| Glyph-level access | Not available | Use CoreText for glyph access |
| Contiguous layout | Viewport-based layout | Major architecture change |

### Critical Migration Note

TextKit 2 does **not** expose glyph-level APIs. If you need individual glyph positions, advances, or custom glyph rendering, you must drop down to CoreText (CTLine/CTRun). This is a deliberate design choice — TextKit 2 abstracts away glyphs to support complex script shaping. See the `coretext` skill for glyph-level work.

## STTextView: Lessons from Production TextKit 2

[STTextView](https://github.com/krzyzanowskim/STTextView) is a production text editor built on TextKit 2 for macOS and iOS. Key architectural lessons:

### Architecture Pattern

STTextView separates concerns through a modular structure:
- **STTextView** — main coordinator (input, delegates, public API)
- **STTextContainerView** — renders text fragments and insertion points
- **STSelectionView** — selection highlight overlays
- **STGutterView** — line numbers and markers
- **STLineHighlightView** — current line highlighting
- Functionality split across extensions (accessibility, copy/paste, find, undo, etc.)

### Plugin System

STTextView uses a plugin architecture for extensibility:

```swift
protocol STPlugin {
    func setUp(context: STPluginContext)
    func tearDown()
}

// Plugins receive context to interact with the text view
// Available plugins: syntax highlighting (TreeSitter), text formation, annotations
textView.addPlugin(myPlugin)
```

### Key Workarounds (from 20+ Apple bug reports)

| Issue | Description | Workaround |
|-------|-------------|------------|
| Selection reset (FB9925647) | `replaceContents` resets multi-cursor selections | Subclass `NSTextContentStorage`, deduplicate selections after replacement |
| Long line jumping | Text view jumps during attribute updates on long lines | Custom viewport layout controller management |
| Missing APIs | `replaceContents()` documented but unavailable in some contexts | Direct `NSAttributedString` manipulation + manual notification |
| Extra line fragments (FB10901256) | Unexpected fragments at document end | Filter anomalous zero-height fragments |
| Chinese IME (FB13789916) | Marked text provides bogus selection ranges | Custom `NSTextInputClient` implementation |
| Selection navigation bugs | Edge cases in cursor movement | Override `NSTextSelectionNavigation` behavior |
| Insertion point not drawn | `drawInsertionPoint` not called reliably | Custom insertion point view (`STInsertionPointView`) |

### When to Use STTextView vs Building Custom

- **Use STTextView** when: building a code editor, need line numbers + syntax highlighting + multi-cursor, want production-tested TextKit 2 workarounds
- **Build custom** when: need very specific text rendering (e.g., chat bubbles, attributed label), don't need full editing, want to avoid GPL license dependency

## Patterns

### ✅ Use TextKit 2 for new text editor projects

```swift
// TextKit 2 is the future — viewport-based layout scales to large documents
let textView = NSTextView(usingTextLayoutManager: true)
```

### ❌ Don't start new projects on TextKit 1 unless you need glyph access

```swift
// TextKit 1 still works but is not receiving new features
let textView = NSTextView(usingTextLayoutManager: false)
```

### ✅ Subclass TextKit 2 components to work around bugs

```swift
// STTextView's approach: strategic subclassing, not full replacement
class STTextContentStorage: NSTextContentStorage {
    override func replaceContents(in range: NSTextRange, with textElements: [NSTextElement]) {
        super.replaceContents(in: range, with: textElements)
        fixSelectionAfterReplacement() // work around FB9925647
    }
}
```

### ❌ Don't assume TextKit 2 APIs work exactly as documented

```swift
// Always test edge cases — documented methods may have subtle bugs
// Especially: selection navigation, long lines, IME input, extra line fragments
```

### ✅ Use the plugin pattern for extensible text editors

```swift
// Separate syntax highlighting, annotations, completions into plugins
textView.addPlugin(SyntaxHighlightPlugin())
textView.addPlugin(AnnotationPlugin())
```

### ❌ Don't put all editor features in a single massive text view subclass

```swift
// This becomes unmaintainable — STTextView uses ~15 extension files to organize
class MyTextView: NSTextView {
    // 3000 lines of mixed concerns
}
```

### ✅ Drop to CoreText for glyph-level rendering

```swift
// TextKit 2 deliberately hides glyphs — use CoreText when you need them
let line = CTLineCreateWithAttributedString(attributedString)
let runs = CTLineGetGlyphRuns(line) as! [CTRun]
```

### ❌ Don't try to access glyphs through TextKit 2

```swift
// TextKit 2 has no glyph API — this is by design, not an omission
// There is no equivalent of NSLayoutManager.glyph(at:)
```
