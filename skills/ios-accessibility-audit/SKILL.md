---
name: ios-accessibility-audit
description: Accessibility audit skill for reviewing iOS/macOS apps against VoiceOver, Dynamic Type, color contrast, and Apple accessibility guidelines. Use when auditing or improving app accessibility.
---

> **First step:** Tell the user: "ios-accessibility-audit skill loaded."

# Accessibility Audit

Comprehensive accessibility audit of iOS/macOS apps against VoiceOver, Dynamic Type, color contrast, Switch Control, and WCAG AA guidelines.

## When This Skill Activates

Use this skill when the user:
- Asks for an accessibility audit or review
- Wants to check VoiceOver support or reading order
- Mentions Dynamic Type compliance or color contrast / WCAG
- Needs to support Switch Control or Full Keyboard Access
- Wants to verify accessibility labels, hints, or traits
- Asks about `UIAccessibility`, `AccessibilityRepresentation`, or a11y

## Audit Process

### Step 1: Discover Files

Use `Grep` to locate accessibility-relevant patterns: `.accessibilityLabel`, `.accessibilityElement`, `Image(` without `.accessibilityHidden`, `.font(.system(size:` (hardcoded sizes), hardcoded `Color(` values.

### Step 2: Audit Categories (in order)

1. VoiceOver  2. Dynamic Type  3. Color & Contrast  4. Motion  5. Switch Control & Keyboard  6. Semantic Structure  7. Images  8. Custom Controls

---

## 1. VoiceOver Audit

```swift
// ✅ Descriptive label and hint
Button(action: addItem) {
    Image(systemName: "plus")
}
.accessibilityLabel("Add item")
.accessibilityHint("Adds a new item to your list")

// ❌ Icon-only button with no label
Button(action: addItem) {
    Image(systemName: "plus")
}

// ✅ Dynamic value for stateful controls
Slider(value: $brightness, in: 0...100)
    .accessibilityValue("\(Int(brightness)) percent")
```

### Traits

```swift
// ✅ Heading trait on section title
Text("Settings")
    .font(.title)
    .accessibilityAddTraits(.isHeader)

// ❌ Custom tap gesture with no button trait
Text("Submit")
    .onTapGesture { submit() }
    // Missing .accessibilityAddTraits(.isButton)
```

### Reading Order and Grouping

```swift
// ✅ Grouped related content
VStack {
    Text(item.name)
    Text(item.price)
}
.accessibilityElement(children: .combine)

// ✅ Explicit reading order override
ZStack {
    backgroundDecoration
    Text("Important announcement")
}
.accessibilityElement(children: .contain)
.accessibilitySortPriority(1)

// ❌ Ungrouped card — VoiceOver reads each element separately
HStack {
    Image(systemName: "star")
    VStack { Text(item.name); Text(item.subtitle) }
    Spacer()
    Text(item.price)
}
```

### AccessibilityRepresentation

```swift
// ✅ Custom control with standard accessibility representation
struct StarRating: View {
    @Binding var rating: Int
    var body: some View {
        HStack {
            ForEach(1...5, id: \.self) { star in
                Image(systemName: star <= rating ? "star.fill" : "star")
                    .onTapGesture { rating = star }
            }
        }
        .accessibilityRepresentation {
            Slider(value: .init(get: { Double(rating) }, set: { rating = Int($0) }),
                   in: 1...5, step: 1)
            .accessibilityLabel("Rating")
            .accessibilityValue("\(rating) out of 5 stars")
        }
    }
}
```

---

## 2. Dynamic Type

```swift
// ✅ System text style scales automatically
Text("Welcome").font(.title)

// ❌ Hardcoded font size
Text("Welcome").font(.system(size: 24))

// ✅ Custom font with scaling
Text("Welcome").font(.custom("Avenir-Heavy", size: 24, relativeTo: .title))
```

### @ScaledMetric

```swift
// ✅ Icon size scales with Dynamic Type
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24
Image(systemName: "heart.fill").frame(width: iconSize, height: iconSize)

// ❌ Fixed icon size ignores Dynamic Type
Image(systemName: "heart.fill").frame(width: 24, height: 24)
```

### Large Text Sizes

```swift
// ✅ Switch layout for accessibility sizes
@Environment(\.dynamicTypeSize) private var typeSize

var body: some View {
    if typeSize.isAccessibilitySize {
        VStack(alignment: .leading) { label; value }
    } else {
        HStack { label; Spacer(); value }
    }
}

// ✅ Scrollable content for large text
ScrollView {
    VStack { Text(longContent).font(.body) }.padding()
}

// ✅ Minimum scale factor as fallback, not primary strategy
Text("Short label").font(.headline).minimumScaleFactor(0.8).lineLimit(2)
```

---

## 3. Color and Contrast

| Element | WCAG AA Minimum |
|---------|----------------|
| Normal text (< 18pt) | 4.5:1 |
| Large text (>= 18pt bold or >= 24pt) | 3:1 |
| UI components and icons | 3:1 |

```swift
// ✅ Semantic colors adapt to light/dark mode
Text("Status").foregroundStyle(.primary)

// ❌ Hardcoded colors with unknown contrast
Text("Status").foregroundColor(Color(red: 0.6, green: 0.6, blue: 0.6)).background(.white)

// ✅ Color + icon for status (not color alone)
HStack {
    Image(systemName: error ? "xmark.circle.fill" : "checkmark.circle.fill")
    Text(error ? "Failed" : "Success")
}
.foregroundStyle(error ? .red : .green)

// ❌ Color is the only indicator
Circle().fill(isOnline ? .green : .red).frame(width: 8, height: 8)

// ✅ Photos opt out of Smart Invert
Image(uiImage: userPhoto).accessibilityIgnoresInvertColors(true)
```

---

## 4. Motion

```swift
// ✅ Check preference before animating
@Environment(\.accessibilityReduceMotion) private var reduceMotion

withAnimation(reduceMotion ? .none : .spring(duration: 0.4)) {
    showDetail.toggle()
}
.transition(reduceMotion ? .opacity : .slide)

// ✅ UIKit reduce motion check
if !UIAccessibility.isReduceMotionEnabled {
    UIView.animate(withDuration: 0.3) {
        view.transform = CGAffineTransform(scaleX: 1.2, y: 1.2)
    }
}

// ❌ Animation ignores user preference
withAnimation(.spring()) { isExpanded.toggle() }
```

---

## 5. Switch Control and Full Keyboard Access

```swift
// ✅ Focusable custom control
struct CustomSlider: View {
    @FocusState private var isFocused: Bool
    var body: some View {
        sliderContent.focusable().focused($isFocused)
    }
}

// ✅ Custom actions for swipe-to-delete patterns
ForEach(items) { item in
    ItemRow(item: item)
        .accessibilityAction(named: "Delete") { deleteItem(item) }
        .accessibilityAction(named: "Mark as favorite") { toggleFavorite(item) }
}

// ✅ Accessible tap action
Text("Play video")
    .onTapGesture { playVideo() }
    .accessibilityAddTraits(.isButton)
    .accessibilityAction { playVideo() }
```

---

## 6. Semantic Structure

```swift
// ✅ Heading traits for section titles
Text("Account Settings").font(.title2).accessibilityAddTraits(.isHeader)

// ✅ Container semantics for logical grouping
VStack { sectionHeader; sectionContent }
    .accessibilityElement(children: .contain)
    .accessibilityLabel("Account Settings section")

// ✅ TabView provides built-in landmark semantics
TabView {
    HomeView().tabItem { Label("Home", systemImage: "house") }
    SettingsView().tabItem { Label("Settings", systemImage: "gear") }
}
```

---

## 7. Images

```swift
// ✅ Informational image with description
Image("product-photo").accessibilityLabel("Red sneakers, side view")

// ✅ Decorative image hidden from VoiceOver
Image("background-gradient").accessibilityHidden(true)

// ❌ Informational image with no label — VoiceOver says "image"
Image("chart-q4-revenue")
```

---

## 8. Custom Controls

```swift
// ✅ Adjustable action for stepper-like controls
struct QuantityPicker: View {
    @Binding var quantity: Int
    var body: some View {
        HStack { Button("-") { quantity -= 1 }; Text("\(quantity)"); Button("+") { quantity += 1 } }
            .accessibilityElement(children: .ignore)
            .accessibilityLabel("Quantity")
            .accessibilityValue("\(quantity)")
            .accessibilityAdjustableAction { direction in
                switch direction {
                case .increment: quantity += 1
                case .decrement: quantity = max(0, quantity - 1)
                @unknown default: break
                }
            }
    }
}

// ✅ Custom scroll action for paged content
PagedCarousel(items: items)
    .accessibilityScrollAction { edge in
        switch edge {
        case .leading, .top: goToPreviousPage()
        case .trailing, .bottom: goToNextPage()
        default: break
        }
    }
```

---

## 9. Testing Tools

### Xcode Accessibility Inspector

1. Xcode > Open Developer Tool > Accessibility Inspector
2. Select simulator/device target
3. **Audit** tab catches common issues automatically
4. **Inspection** pointer verifies labels, traits, and frames per element

### VoiceOver Testing Protocol

1. Enable VoiceOver (Settings > Accessibility > VoiceOver)
2. Swipe right through every screen element
3. Verify: meaningful label, correct trait, logical order
4. Test rotor navigation: headings, links, form controls
5. Confirm all actions reachable without gestures alone

### Automated Audit (Xcode 15+)

```swift
func testAccessibility() throws {
    let app = XCUIApplication()
    app.launch()

    try app.performAccessibilityAudit()

    try app.performAccessibilityAudit(for: [.dynamicType, .contrast, .sufficientElementDescription])

    try app.performAccessibilityAudit(for: .all) { issue in
        issue.element?.label == "decorative-divider" // return true to ignore
    }
}
```

---

## Common Issues Table

| Issue | Severity | What to Check |
|-------|----------|---------------|
| Missing `accessibilityLabel` on icon-only buttons | 🔴 Critical | All `Button`/tappable views with no visible text |
| Hardcoded font sizes (`.system(size:)`) | 🔴 Critical | Every `.font()` modifier |
| Color-only status indicators | 🔴 Critical | Error states, online/offline, status badges |
| Missing heading traits on section titles | 🟠 High | `.title`/`.headline` text acting as headings |
| Animations ignoring Reduce Motion | 🟠 High | All `withAnimation`/`.animation` calls |
| Ungrouped card content | 🟠 High | List rows, cards, composite views |
| Informational images without labels | 🟠 High | All `Image()` that convey meaning |
| Tap targets below 44pt | 🟡 Medium | Custom buttons, small icon buttons |
| Missing `accessibilityHint` on ambiguous actions | 🟡 Medium | Buttons with unclear labels |
| Missing `accessibilityIgnoresInvertColors` on photos | 🟡 Medium | User photos, product images |
| Decorative images not hidden | 🟢 Low | Background images, separators |
| Redundant labels matching visible text | 🟢 Low | Labels duplicating SwiftUI defaults |

## Output Format

```
## Accessibility Audit: [FileName].swift

### 🔴 Critical
1. Line 42: Icon-only button missing accessibilityLabel
   Fix: Add .accessibilityLabel("Delete item")

### 🟠 High
1. Line 78: Section title "Recent" missing heading trait
   Fix: Add .accessibilityAddTraits(.isHeader)

### 🟡 Medium
1. Line 105: Custom button frame is 30x30, below 44pt minimum
   Fix: Add .frame(minWidth: 44, minHeight: 44)

### 🟢 Low
1. Line 12: Decorative divider not hidden from VoiceOver
   Fix: Add .accessibilityHidden(true)

### ✅ Strengths
- Good use of semantic colors throughout
- Dynamic Type supported on all text elements
- Logical VoiceOver reading order in main content
```
