---
name: swiftui-custom-layout
description: SwiftUI Layout protocol for custom container layouts including flow layouts, radial layouts, and animated transitions. Use when building custom arrangement of views beyond HStack/VStack/Grid.
---

# SwiftUI Custom Layout

## When This Skill Activates

Use this skill when the user:
- Wants to arrange views in a custom pattern (flow, radial, masonry, etc.)
- Asks about the `Layout` protocol, `sizeThatFits`, or `placeSubviews`
- Needs a tag/chip layout that wraps items to the next line
- Wants a circular or radial arrangement of views
- Asks about `AnyLayout` or animating between different layouts
- Needs per-subview layout configuration via `LayoutValueKey`
- Wants to optimize layout with caching (`makeCache`, `updateCache`)
- Asks about `ViewThatFits` for adaptive layouts
- Asks about `ProposedViewSize` or `ViewDimensions`
- Needs to go beyond `HStack`, `VStack`, `LazyVGrid`, or `Grid`

## Decision Tree

```
What layout behavior do you need?
|
+-- Simple stacking -----------> HStack / VStack / LazyHStack / LazyVStack
+-- Fixed grid ----------------> LazyVGrid / LazyHGrid / Grid (iOS 16+)
+-- Wrapping flow/tags --------> Layout protocol (FlowLayout)
+-- Circular/radial -----------> Layout protocol (RadialLayout)
+-- Adaptive per space --------> ViewThatFits or AnyLayout
+-- Animate layout switch -----> AnyLayout + withAnimation
+-- Need child size first -----> Layout protocol (never GeometryReader)
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `Layout` protocol | iOS 16 / macOS 13 | Custom container layouts |
| `ProposedViewSize` | iOS 16 / macOS 13 | Width/height proposals |
| `LayoutSubview` | iOS 16 / macOS 13 | Proxy for each child |
| `ViewDimensions` | iOS 16 / macOS 13 | Measured size + alignments |
| `LayoutValueKey` | iOS 16 / macOS 13 | Per-subview metadata |
| `AnyLayout` | iOS 16 / macOS 13 | Type-erased layout for animation |
| `ViewThatFits` | iOS 16 / macOS 13 | Picks first fitting layout |

## The Layout Protocol

Every custom layout implements two required methods:

```swift
struct MyLayout: Layout {
    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        // Return the container's ideal size given the proposal
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize,
                       subviews: Subviews, cache: inout ()) {
        // Position each subview within bounds
    }
}
```

**ProposedViewSize** -- the system proposes a size to your layout. Access `.width`/`.height` as optionals:
- `.zero` -- measure minimum | `.infinity` -- measure ideal/max | `.unspecified` -- both nil

**ViewDimensions** -- call `subview.dimensions(in: proposal)` to get measured size plus alignment guide values like `dimensions[.firstTextBaseline]`.

## Flow Layout (Wrapping Tags)

A complete flow layout that wraps items to the next line when they exceed available width:

```swift
struct FlowLayout: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize {
        let sizes = subviews.map { $0.sizeThatFits(.unspecified) }
        return computeLayout(sizes: sizes, containerWidth: proposal.width ?? .infinity).size
    }

    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) {
        let sizes = subviews.map { $0.sizeThatFits(.unspecified) }
        let offsets = computeLayout(sizes: sizes, containerWidth: bounds.width).offsets

        for (index, subview) in subviews.enumerated() {
            subview.place(
                at: CGPoint(
                    x: bounds.minX + offsets[index].x,
                    y: bounds.minY + offsets[index].y
                ),
                proposal: .unspecified
            )
        }
    }

    private func computeLayout(
        sizes: [CGSize],
        containerWidth: CGFloat
    ) -> (offsets: [CGPoint], size: CGSize) {
        var offsets: [CGPoint] = []
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var lineHeight: CGFloat = 0
        var maxWidth: CGFloat = 0

        for size in sizes {
            if currentX + size.width > containerWidth, currentX > 0 {
                currentX = 0
                currentY += lineHeight + spacing
                lineHeight = 0
            }
            offsets.append(CGPoint(x: currentX, y: currentY))
            lineHeight = max(lineHeight, size.height)
            currentX += size.width + spacing
            maxWidth = max(maxWidth, currentX - spacing)
        }

        return (offsets, CGSize(width: maxWidth, height: currentY + lineHeight))
    }
}

// Usage
FlowLayout(spacing: 8) {
    ForEach(["Swift", "SwiftUI", "Layout", "Custom", "Flow"], id: \.self) { tag in
        Text(tag)
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(.blue.opacity(0.15), in: Capsule())
    }
}
```

## Radial Layout

```swift
struct RadialLayout: Layout {
    var radius: CGFloat = 100

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        CGSize(width: radius * 2 + 50, height: radius * 2 + 50)
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize,
                       subviews: Subviews, cache: inout ()) {
        guard !subviews.isEmpty else { return }
        let step = Angle.degrees(360.0 / Double(subviews.count)).radians
        for (index, subview) in subviews.enumerated() {
            let angle = step * Double(index) - .pi / 2
            subview.place(
                at: CGPoint(x: bounds.midX + radius * cos(angle),
                             y: bounds.midY + radius * sin(angle)),
                anchor: .center, proposal: .unspecified
            )
        }
    }
}
```

## LayoutValueKey (Per-Subview Configuration)

Pass metadata from individual subviews to the layout:

```swift
struct LayoutPriority: LayoutValueKey {
    static let defaultValue: Double = 0
}

extension View {
    func customPriority(_ value: Double) -> some View {
        layoutValue(key: LayoutPriority.self, value: value)
    }
}

// Read inside placeSubviews
let sorted = subviews.sorted { $0[LayoutPriority.self] > $1[LayoutPriority.self] }

// Usage
FlowLayout {
    Text("Important").customPriority(10)
    Text("Normal").customPriority(0)
}
```

## Cache for Performance

Avoid redundant measurements by caching. Change the cache type from `()` to a custom struct:

```swift
struct CachedFlowLayout: Layout {
    var spacing: CGFloat = 8

    struct CacheData {
        var sizes: [CGSize] = []
    }

    func makeCache(subviews: Subviews) -> CacheData {
        CacheData(sizes: subviews.map { $0.sizeThatFits(.unspecified) })
    }

    func updateCache(_ cache: inout CacheData, subviews: Subviews) {
        cache.sizes = subviews.map { $0.sizeThatFits(.unspecified) }
    }

    func sizeThatFits(
        proposal: ProposedViewSize, subviews: Subviews, cache: inout CacheData
    ) -> CGSize {
        computeLayout(sizes: cache.sizes, containerWidth: proposal.width ?? .infinity).size
    }

    func placeSubviews(
        in bounds: CGRect, proposal: ProposedViewSize,
        subviews: Subviews, cache: inout CacheData
    ) {
        let offsets = computeLayout(sizes: cache.sizes, containerWidth: bounds.width).offsets
        for (index, subview) in subviews.enumerated() {
            subview.place(
                at: CGPoint(x: bounds.minX + offsets[index].x, y: bounds.minY + offsets[index].y),
                proposal: .unspecified
            )
        }
    }

    // Reuse the same computeLayout helper as FlowLayout
    private func computeLayout(sizes: [CGSize], containerWidth: CGFloat)
        -> (offsets: [CGPoint], size: CGSize) { /* ... */ }
}
```

## Animating Between Layouts with AnyLayout

Wrap layouts in `AnyLayout` to animate transitions between different layout types:

```swift
@State private var useFlow = false

var body: some View {
    let layout = useFlow ? AnyLayout(FlowLayout(spacing: 10))
                         : AnyLayout(HStackLayout(spacing: 10))
    layout {
        ForEach(0..<5) { i in
            Circle().fill(.blue).frame(width: 40, height: 40)
        }
    }
    .animation(.spring, value: useFlow)

    Toggle("Flow Layout", isOn: $useFlow)
}
```

## Layout Composition

Custom layouts compose naturally -- nest them like any SwiftUI container:

```swift
FlowLayout(spacing: 12) {
    RadialLayout(radius: 60) {
        ForEach(0..<6) { _ in
            Circle().fill(.mint).frame(width: 20, height: 20)
        }
    }
    .frame(width: 160, height: 160)

    FlowLayout(spacing: 6) {
        ForEach(["A", "B", "C"], id: \.self) { tag in
            Text(tag).padding(8).background(.orange.opacity(0.2), in: Capsule())
        }
    }
}
```

## ViewThatFits

SwiftUI measures each child and picks the first one that fits the available space:

```swift
ViewThatFits {
    // First choice: labels with text
    HStack(spacing: 16) {
        Label("Copy", systemImage: "doc.on.doc")
        Label("Paste", systemImage: "doc.on.clipboard")
        Label("Delete", systemImage: "trash")
    }
    // Fallback: icons only
    HStack(spacing: 16) {
        Image(systemName: "doc.on.doc")
        Image(systemName: "doc.on.clipboard")
        Image(systemName: "trash")
    }
    // Last resort: overflow menu
    Menu("Actions") {
        Button("Copy", action: {}); Button("Paste", action: {}); Button("Delete", action: {})
    }
}
```

## Patterns

### ✅ Good Patterns

```swift
// ✅ Use Layout protocol for custom arrangements
FlowLayout(spacing: 8) {
    ForEach(items) { item in ItemView(item: item) }
}

// ✅ Cache measurements for layouts with many subviews
func makeCache(subviews: Subviews) -> CacheData {
    CacheData(sizes: subviews.map { $0.sizeThatFits(.unspecified) })
}

// ✅ Animate layout changes with AnyLayout
let layout = condition ? AnyLayout(VStackLayout()) : AnyLayout(HStackLayout())
layout { content }.animation(.spring, value: condition)

// ✅ Use ViewThatFits for responsive alternatives
ViewThatFits { ExpandedView(); CompactView() }

// ✅ Use LayoutValueKey for per-child configuration
Text("VIP").layoutValue(key: Priority.self, value: 10)
```

### ❌ Bad Patterns

```swift
// ❌ Don't use GeometryReader for layout -- it greedily fills space and breaks sizing
GeometryReader { geo in
    HStack { ForEach(items) { item in ItemView().frame(width: geo.size.width / 3) } }
}

// ❌ Don't re-measure in both methods without a cache
func sizeThatFits(...) { subviews.map { $0.sizeThatFits(.unspecified) } } // measured
func placeSubviews(...) { subviews.map { $0.sizeThatFits(.unspecified) } } // again!

// ❌ Don't ignore the proposal
func sizeThatFits(proposal: ProposedViewSize, ...) -> CGSize {
    CGSize(width: 500, height: 500) // Hardcoded, ignores container
}

// ❌ Don't forget empty subviews guard
let step = 360.0 / Double(subviews.count) // Division by zero
```
