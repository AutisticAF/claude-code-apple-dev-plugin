---
name: coretext
description: CoreText patterns for low-level text layout, custom font handling, glyph-level rendering, and typographic features. Use when you need precise control over text rendering below TextKit.
---

# CoreText

CoreText is Apple's low-level text layout and font engine sitting directly above Core Graphics. It is the foundation that TextKit 1, TextKit 2, and SwiftUI Text are all built upon. Use CoreText when you need maximum control over how text is shaped, measured, and drawn.

## When This Skill Activates

- User needs glyph-level control over text rendering
- User is drawing text directly into a CGContext (custom views, PDF generation)
- User needs OpenType font features (small caps, stylistic alternates, ligatures, old-style figures)
- User is working with CTFrame, CTLine, CTRun, or CTFramesetter
- User needs hit testing or caret positioning at the glyph level
- User needs performance-critical read-only text layout (thousands of labels, chat bubbles)
- User needs variable font axis control or vertical text layout for CJK

## Decision Tree

```
What text rendering do you need?
|
+-- Display styled text (standard controls)
|   +-- SwiftUI? → Text with AttributedString
|   +-- UIKit? → UILabel / UITextView
|
+-- Editable text → TextKit 2 or TextKit 1
|
+-- Read-only custom rendering
|   +-- Multi-line frame layout → CTFramesetter / CTFrame
|   +-- Single-line measurement → CTLine
|   +-- Glyph iteration → CTRun
|
+-- Font feature access → CTFontDescriptor with feature settings
|
+-- Maximum performance (thousands of items) → CoreText directly
```

## API Availability

| API | iOS | macOS | Notes |
|-----|-----|-------|-------|
| `CTFont` | 3.2+ | 10.5+ | Core font object |
| `CTFontDescriptor` | 3.2+ | 10.5+ | Font matching and features |
| `CTFramesetter` / `CTFrame` | 3.2+ | 10.5+ | Multi-line layout |
| `CTLine` / `CTRun` | 3.2+ | 10.5+ | Line and glyph-run access |
| `CTTypesetter` | 3.2+ | 10.5+ | Low-level line breaking |
| Variable font axes | 11.0+ | 10.13+ | `kCTFontVariationAttribute` |

CoreText is extremely stable. APIs from iOS 3.2 / macOS 10.5 still work unchanged. Swift overlay improvements (toll-free bridging, nullability) arrived gradually through iOS 11-15.

## CTFont — Creating and Querying Fonts

```swift
import CoreText

// From a font name
let font = CTFontCreateWithName("Helvetica Neue" as CFString, 16.0, nil)

// System UI font for a specific language
let jaFont = CTFontCreateUIFontForLanguage(.system, 14.0, "ja" as CFString)!

// From a CGFont (e.g., loaded from raw data)
let cgFont = CGFont(dataProvider)!
let ctFont = CTFontCreateWithGraphicsFont(cgFont, 16.0, nil, nil)

// Bold variant of an existing font
let boldFont = CTFontCreateCopyWithSymbolicTraits(font, 0.0, nil, .boldTrait, .boldTrait)

// Metrics
let ascent = CTFontGetAscent(font)
let descent = CTFontGetDescent(font)
let leading = CTFontGetLeading(font)
let lineHeight = ascent + descent + leading
let boundingBox = CTFontGetBoundingBox(font)
let capHeight = CTFontGetCapHeight(font)
let xHeight = CTFontGetXHeight(font)
```

### Variable Font Axes

```swift
if let axes = CTFontCopyVariationAxes(font) as? [[CFString: Any]] {
    for axis in axes {
        let name = axis[kCTFontVariationAxisNameKey]
        let min = axis[kCTFontVariationAxisMinimumValueKey]
        let max = axis[kCTFontVariationAxisMaximumValueKey]
        print("\(name!): \(min!)–\(max!)")
    }
}

// Create font with specific axis values (e.g., weight 700)
let variations: [CFString: CGFloat] = [
    CFString(kCTFontVariationAxisIdentifierKey): 700
]
let desc = CTFontDescriptorCreateWithAttributes([
    kCTFontVariationAttribute: variations
] as CFDictionary)
let variableFont = CTFontCreateWithFontDescriptor(desc, 16.0, nil)
```

## CTFontDescriptor — Font Matching and Features

```swift
// Create a descriptor and match
let attrs: [CFString: Any] = [
    kCTFontFamilyNameAttribute: "Helvetica Neue",
    kCTFontTraitsAttribute: [kCTFontWeightTrait: 0.4]
]
let descriptor = CTFontDescriptorCreateWithAttributes(attrs as CFDictionary)
let matched = CTFontDescriptorCreateMatchingFontDescriptor(
    descriptor, [kCTFontFamilyNameAttribute, kCTFontTraitsAttribute] as CFSet
)
```

### Font Feature Settings (Small Caps, Old-Style Figures, Alternates)

```swift
let smallCaps: [CFString: Any] = [
    kCTFontFeatureTypeIdentifierKey: kLetterCaseType,
    kCTFontFeatureSelectorIdentifierKey: kSmallCapsSelector
]
let oldStyleFigures: [CFString: Any] = [
    kCTFontFeatureTypeIdentifierKey: kNumberCaseType,
    kCTFontFeatureSelectorIdentifierKey: kLowerCaseNumbersSelector
]
let contextualAlts: [CFString: Any] = [
    kCTFontFeatureTypeIdentifierKey: kContextualAlternatesType,
    kCTFontFeatureSelectorIdentifierKey: kContextualAlternatesOnSelector
]

let featureAttrs: [CFString: Any] = [
    kCTFontNameAttribute: "Helvetica Neue" as CFString,
    kCTFontFeatureSettingsAttribute: [smallCaps, oldStyleFigures, contextualAlts]
]
let featureDesc = CTFontDescriptorCreateWithAttributes(featureAttrs as CFDictionary)
let styledFont = CTFontCreateWithFontDescriptor(featureDesc, 14.0, nil)
```

### Font Cascade (Fallback) Lists

```swift
let cascadeList = CTFontCopyDefaultCascadeListForLanguages(
    font, ["zh-Hans", "ja"] as CFArray
)
// Returns CTFontDescriptors for fallback fonts
```

## The Layout Pipeline: Framesetter -> Frame -> Line -> Run

```
NSAttributedString
       |
CTFramesetterCreateWithAttributedString(_:)
       |
CTFramesetter --> CTFramesetterCreateFrame(_:_:_:_:) + CGPath
       |
CTFrame --> CTFrameGetLines(_:) --> [CTLine]
                                       |
                                CTLineGetGlyphRuns(_:) --> [CTRun]
                                                             |
                                                      CGGlyph + CGPoint pairs
```

### Complete Layout Example

```swift
import CoreText
import CoreGraphics

func layoutTextInPath(text: NSAttributedString, path: CGPath, context: CGContext) {
    let framesetter = CTFramesetterCreateWithAttributedString(text)
    let frame = CTFramesetterCreateFrame(
        framesetter, CFRange(location: 0, length: 0), path, nil
    )

    // Flip coordinate system (CoreText uses bottom-left origin)
    context.saveGState()
    context.translateBy(x: 0, y: path.boundingBox.maxY)
    context.scaleBy(x: 1.0, y: -1.0)
    CTFrameDraw(frame, context)
    context.restoreGState()
}
```

## CTLine — Single-Line Layout and Truncation

```swift
let attrs: [NSAttributedString.Key: Any] = [
    .font: CTFontCreateWithName("Helvetica" as CFString, 16.0, nil),
    .foregroundColor: CGColor(red: 0, green: 0, blue: 0, alpha: 1)
]
let line = CTLineCreateWithAttributedString(
    NSAttributedString(string: "Hello, CoreText!", attributes: attrs)
)

// Typographic bounds
var ascent: CGFloat = 0, descent: CGFloat = 0, leading: CGFloat = 0
let width = CTLineGetTypographicBounds(line, &ascent, &descent, &leading)

// Image bounds (tight box around actual glyphs) and optical bounds
let imageBounds = CTLineGetImageBounds(line, context)
let opticalBounds = CTLineGetBoundsWithOptions(line, [.useOpticalBounds])

// Truncation
let ellipsis = CTLineCreateWithAttributedString(
    NSAttributedString(string: "\u{2026}", attributes: attrs)
)
let truncated = CTLineCreateTruncatedLine(line, 200.0, .end, ellipsis)
```

## CTRun — Glyph-Level Access

A CTRun is a contiguous sequence of glyphs sharing the same attributes.

```swift
let runs = CTLineGetGlyphRuns(line) as! [CTRun]
for run in runs {
    let count = CTRunGetGlyphCount(run)
    var glyphs = [CGGlyph](repeating: 0, count: count)
    var positions = [CGPoint](repeating: .zero, count: count)
    var advances = [CGSize](repeating: .zero, count: count)

    CTRunGetGlyphs(run, CFRange(location: 0, length: 0), &glyphs)
    CTRunGetPositions(run, CFRange(location: 0, length: 0), &positions)
    CTRunGetAdvances(run, CFRange(location: 0, length: 0), &advances)

    let runAttrs = CTRunGetAttributes(run) as! [NSAttributedString.Key: Any]
    let stringRange = CTRunGetStringRange(run)
}
```

## Drawing Text in a Custom UIView

```swift
import UIKit
import CoreText

final class CoreTextView: UIView {
    var attributedText: NSAttributedString? { didSet { setNeedsDisplay() } }

    override func draw(_ rect: CGRect) {
        guard let ctx = UIGraphicsGetCurrentContext(),
              let text = attributedText else { return }

        ctx.saveGState()
        ctx.translateBy(x: 0, y: bounds.height)
        ctx.scaleBy(x: 1.0, y: -1.0)

        let insets = UIEdgeInsets(top: 8, left: 12, bottom: 8, right: 12)
        let path = CGPath(rect: bounds.inset(by: insets), transform: nil)
        let framesetter = CTFramesetterCreateWithAttributedString(text)
        let frame = CTFramesetterCreateFrame(
            framesetter, CFRange(location: 0, length: 0), path, nil
        )
        CTFrameDraw(frame, ctx)
        ctx.restoreGState()
    }
}

// For per-line control (custom spacing, backgrounds), draw lines individually:
func drawLinesIndividually(frame: CTFrame, in ctx: CGContext, rect: CGRect) {
    let lines = CTFrameGetLines(frame) as! [CTLine]
    var origins = [CGPoint](repeating: .zero, count: lines.count)
    CTFrameGetLineOrigins(frame, CFRange(location: 0, length: 0), &origins)
    ctx.saveGState()
    ctx.translateBy(x: 0, y: rect.height)
    ctx.scaleBy(x: 1.0, y: -1.0)
    for (i, line) in lines.enumerated() {
        ctx.textPosition = origins[i]
        CTLineDraw(line, ctx)
    }
    ctx.restoreGState()
}
```

## Hit Testing and Caret Positioning

```swift
func stringIndex(at point: CGPoint, in frame: CTFrame, frameRect: CGRect) -> CFIndex {
    let lines = CTFrameGetLines(frame) as! [CTLine]
    var origins = [CGPoint](repeating: .zero, count: lines.count)
    CTFrameGetLineOrigins(frame, CFRange(location: 0, length: 0), &origins)

    // Convert from UIKit (top-left origin) to CoreText (bottom-left origin)
    let ctPoint = CGPoint(x: point.x, y: frameRect.height - point.y)

    for (i, line) in lines.enumerated() {
        var ascent: CGFloat = 0, descent: CGFloat = 0
        CTLineGetTypographicBounds(line, &ascent, &descent, nil)
        if ctPoint.y <= origins[i].y + ascent && ctPoint.y >= origins[i].y - descent {
            return CTLineGetStringIndexForPosition(
                line, CGPoint(x: ctPoint.x - origins[i].x, y: 0)
            )
        }
    }
    return kCFNotFound
}

// Caret X offset for a given string index
func caretOffset(forIndex index: CFIndex, in line: CTLine) -> CGFloat {
    CTLineGetOffsetForStringIndex(line, index, nil)
}
```

## Vertical Text Layout (CJK)

```swift
let frameAttrs: [CFString: Any] = [
    kCTFrameProgressionAttributeName: CTFrameProgression.rightToLeft.rawValue
]
let frame = CTFramesetterCreateFrame(
    framesetter, CFRange(location: 0, length: 0), path, frameAttrs as CFDictionary
)
// Lines flow right-to-left; glyphs render vertically
```

## Performance

CoreText is the fastest text rendering path on Apple platforms. Use it when:

- **Rendering thousands of items** -- chat bubbles, spreadsheets, data grids
- **PDF generation** -- CTFrameDraw/CTLineDraw write directly into a PDF CGContext
- **Offscreen rendering** -- drawing into bitmap contexts for caching or compositing
- **Read-only display** -- CoreText has no editing infrastructure, but that makes it leaner

Do NOT use CoreText when you need text editing (TextKit), standard controls suffice (UILabel), or you need built-in accessibility text interaction.

## Patterns

### Memory Management

✅ Let Swift manage CT object lifetimes (CF types are ARC-bridged)

```swift
let font = CTFontCreateWithName("Helvetica" as CFString, 16.0, nil)
// Automatically released when it goes out of scope
```

❌ Manual CFRelease in Swift (unnecessary, risks double-free)

```swift
let font = CTFontCreateWithName("Helvetica" as CFString, 16.0, nil)
CFRelease(font)  // Swift already manages this
```

### Coordinate System

✅ Always flip before drawing

```swift
context.translateBy(x: 0, y: rect.height)
context.scaleBy(x: 1.0, y: -1.0)
CTFrameDraw(frame, context)
```

❌ Drawing without flipping (text renders upside-down)

```swift
CTFrameDraw(frame, context)  // Upside-down text
```

### Right Level of Abstraction

✅ Use CoreText only when you need its power (glyph control, custom rendering, performance). For simple styled labels, just use UILabel.

❌ Reimplementing UILabel with hundreds of lines of CoreText -- wasteful and loses built-in accessibility.

### Attribute Keys

✅ Use CTFont objects in CoreText pipelines. ❌ Mixing UIFont into pure CT code can cause subtle issues -- prefer `CTFontCreateWithName` over `UIFont.systemFont(ofSize:)`.

### Always Measure Before Layout

✅ Use CTFramesetterSuggestFrameSizeWithConstraints

```swift
let fitSize = CTFramesetterSuggestFrameSizeWithConstraints(
    framesetter, CFRange(location: 0, length: 0), nil,
    CGSize(width: maxWidth, height: .greatestFiniteMagnitude), nil
)
```

❌ Guessing frame size (text gets clipped)

```swift
let path = CGPath(rect: CGRect(x: 0, y: 0, width: 200, height: 100), transform: nil)
// 100pt may not be enough -- always measure first
```
