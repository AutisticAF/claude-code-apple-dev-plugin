---
name: swiftui-presentations
description: SwiftUI presentation patterns including sheets, fullScreenCover, popover, inspector, alert, confirmationDialog, and presentation customization. Use when presenting modal or overlay content.
---

> **First step:** Tell the user: "swiftui-presentations skill loaded."

# SwiftUI Presentations

Modal and overlay content patterns for SwiftUI apps.

## When This Skill Activates

Use this skill when the user:
- Wants to present a sheet, modal, or overlay
- Asks about fullScreenCover, popover, inspector, alert, or confirmationDialog
- Asks about presentation detents, dismissal behavior, or passing data to presented views

## Decision Tree

```
What kind of content are you presenting?
|
+- Contextual info or form (non-blocking)
|  +- iPhone -> .sheet (auto adapts to full height)
|  +- iPad/Mac, anchored to element -> .popover
|  +- iPad/Mac, side panel -> .inspector
|
+- Immersive flow (onboarding, login, full editor)
|  +- .fullScreenCover
|
+- Destructive or irreversible action confirmation
|  +- .confirmationDialog
|
+- Short message with simple actions (OK, Cancel)
|  +- .alert
|
+- Navigating to a new screen in a stack
|  +- NavigationLink (not a presentation)
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `.sheet(isPresented:)` | iOS 14 | Boolean-driven sheet |
| `.sheet(item:)` | iOS 14 | Item-driven sheet |
| `.fullScreenCover(isPresented:)` | iOS 14 | Full-screen modal |
| `.fullScreenCover(item:)` | iOS 14 | Item-driven full-screen modal |
| `.popover(isPresented:)` | iOS 14 | Popover (iPad/Mac; sheet on iPhone) |
| `.alert(title:isPresented:)` | iOS 15 | Alert with actions and message |
| `.confirmationDialog(title:isPresented:)` | iOS 15 | Action sheet replacement |
| `.presentationDetents()` | iOS 16 | Half-sheet and custom heights |
| `.presentationDragIndicator()` | iOS 16 | Show/hide grab handle |
| `.presentationCornerRadius()` | iOS 16.4 | Custom corner radius |
| `.presentationContentInteraction()` | iOS 16.4 | Scroll vs. resize priority |
| `.presentationBackgroundInteraction()` | iOS 16.4 | Interact with content behind sheet |
| `.interactiveDismissDisabled()` | iOS 15 | Prevent swipe-to-dismiss |
| `.inspector(isPresented:)` | iOS 17 | Side panel inspector |

## Sheet Presentations

### Boolean-Driven Sheet

```swift
@State private var showSettings = false

Button("Settings") { showSettings = true }
    .sheet(isPresented: $showSettings) {
        SettingsView()
    }
```

### Item-Driven Sheet

The sheet appears when the item is non-nil. The item must conform to `Identifiable`.

```swift
@State private var selectedItem: Item?

List(items) { item in
    Button(item.name) { selectedItem = item }
}
.sheet(item: $selectedItem) { item in
    ItemDetailView(item: item)
}
```

## Full-Screen Cover

Use for immersive flows. No swipe-to-dismiss gesture; slides up from the bottom.

```swift
@State private var showOnboarding = true

MainView()
    .fullScreenCover(isPresented: $showOnboarding) {
        OnboardingFlow()
    }
```

## Popover

Floating panel on iPad/Mac; adapts to sheet on iPhone.

```swift
@State private var showFilter = false

Button("Filter") { showFilter = true }
    .popover(isPresented: $showFilter,
             attachmentAnchor: .point(.bottom),
             arrowEdge: .top) {
        FilterView()
            .frame(minWidth: 300, minHeight: 400)
    }
```

- `attachmentAnchor`: Where on the source view the popover attaches (`.point(.top)`, `.rect(.bounds)`)
- `arrowEdge`: Which edge of the popover the arrow appears on

## Inspector (iOS 17+)

Side panel on iPad/Mac; sheet on iPhone.

```swift
@State private var showInspector = false

EditorView()
    .inspector(isPresented: $showInspector) {
        InspectorContent()
            .inspectorColumnWidth(min: 250, ideal: 300, max: 400)
    }
```

## Presentation Detents

Control the height stops for a sheet. Available from iOS 16+.

```swift
.sheet(isPresented: $showPanel) {
    PanelView()
        .presentationDetents([.medium, .large])
}
```

### Built-In and Custom Detents

```swift
// Fixed height
.presentationDetents([.height(200)])

// Fraction of screen
.presentationDetents([.fraction(0.25), .large])

// Custom detent with context
struct MyCustomDetent: CustomPresentationDetent {
    static func height(in context: Context) -> CGFloat? {
        max(context.maxDetentValue * 0.3, 200)
    }
}

// Usage
.presentationDetents([.custom(MyCustomDetent.self), .large])
```

### Tracking the Active Detent

```swift
@State private var selectedDetent: PresentationDetent = .medium

.sheet(isPresented: $showPanel) {
    PanelView()
        .presentationDetents([.medium, .large], selection: $selectedDetent)
}
```

## Presentation Customization Modifiers

Apply inside the presented view:

```swift
ScrollView { LongFormContent() }
    .presentationDetents([.medium, .large])
    .presentationDragIndicator(.visible)
    .presentationCornerRadius(20)
    .presentationContentInteraction(.scrolls)
    .presentationBackgroundInteraction(.enabled(upThrough: .medium))
```

| Modifier | Purpose |
|----------|---------|
| `.presentationDragIndicator(.visible)` | Show/hide the grab handle |
| `.presentationCornerRadius(20)` | Custom corner radius (iOS 16.4+) |
| `.presentationContentInteraction(.scrolls)` | Prioritize scrolling over sheet resizing |
| `.presentationBackgroundInteraction(.enabled)` | Allow interaction with content behind sheet |

## Alerts

```swift
@State private var showAlert = false

Button("Delete", role: .destructive) { showAlert = true }
    .alert("Delete Item?", isPresented: $showAlert) {
        Button("Delete", role: .destructive) { performDelete() }
        Button("Cancel", role: .cancel) { }
    } message: {
        Text("This action cannot be undone.")
    }
```

## Confirmation Dialog

Use instead of alert when offering multiple actions. Renders as action sheet on iPhone, popover on iPad/Mac.

```swift
.confirmationDialog("Change Photo", isPresented: $showDialog, titleVisibility: .visible) {
    Button("Take Photo") { takePhoto() }
    Button("Choose from Library") { chooseFromLibrary() }
    Button("Remove Photo", role: .destructive) { removePhoto() }
} message: {
    Text("Select a new profile photo.")
}
```

## Dismissing Presentations

### Using @Environment(\.dismiss)

```swift
struct DetailSheet: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form { /* ... */ }
                .toolbar {
                    ToolbarItem(placement: .confirmationAction) {
                        Button("Done") { dismiss() }
                    }
                }
        }
    }
}
```

### Using the isPresented Binding

```swift
struct ChildView: View {
    @Binding var isPresented: Bool
    var body: some View {
        Button("Close") { isPresented = false }
    }
}
```

## Preventing Dismissal

Block swipe-to-dismiss while allowing programmatic dismissal. Pass `true` to always block, or a condition.

```swift
@State private var hasUnsavedChanges = false

Form { /* ... */ }
    .interactiveDismissDisabled(hasUnsavedChanges)
```

## Good and Bad Patterns

```swift
// ❌ Multiple sheets on the same view — only the last one wins
VStack {
    Text("Hello")
}
.sheet(isPresented: $showA) { ViewA() }
.sheet(isPresented: $showB) { ViewB() }

// ✅ Attach each sheet to a different child view, or use item-based presentation
VStack {
    Button("A") { showA = true }
        .sheet(isPresented: $showA) { ViewA() }
    Button("B") { showB = true }
        .sheet(isPresented: $showB) { ViewB() }
}
```

```swift
// ❌ Setting item inside the sheet's onDismiss — causes re-presentation loop
.sheet(item: $selectedItem, onDismiss: { selectedItem = anotherItem }) {
    DetailView(item: $0)
}

// ✅ Use onDismiss only for cleanup, not for triggering new presentations
.sheet(item: $selectedItem, onDismiss: { cleanUpResources() }) {
    DetailView(item: $0)
}
```

```swift
// ❌ Forgetting Identifiable on the item type
struct Task {              // No Identifiable conformance
    let name: String
}
// .sheet(item:) won't compile

// ✅ Conform to Identifiable
struct Task: Identifiable {
    let id = UUID()
    let name: String
}
```

```swift
// ❌ Calling dismiss() on the parent instead of inside the sheet
struct Parent: View {
    @Environment(\.dismiss) private var dismiss
    @State private var showSheet = false

    var body: some View {
        Button("Show") { showSheet = true }
            .sheet(isPresented: $showSheet) {
                Button("Close") { dismiss() }  // Dismisses Parent, not the sheet
            }
    }
}

// ✅ Access @Environment(\.dismiss) inside the presented view
struct SheetContent: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        Button("Close") { dismiss() }  // Correctly dismisses the sheet
    }
}
```

## References

- [SwiftUI sheet(isPresented:)](https://developer.apple.com/documentation/swiftui/view/sheet(ispresented:ondismiss:content:))
- [SwiftUI fullScreenCover](https://developer.apple.com/documentation/swiftui/view/fullscreencover(ispresented:ondismiss:content:))
- [SwiftUI presentationDetents](https://developer.apple.com/documentation/swiftui/view/presentationdetents(_:))
- [SwiftUI inspector](https://developer.apple.com/documentation/swiftui/view/inspector(ispresented:content:))
- [SwiftUI alert](https://developer.apple.com/documentation/swiftui/view/alert(_:ispresented:actions:message:))
- [SwiftUI confirmationDialog](https://developer.apple.com/documentation/swiftui/view/confirmationdialog(_:ispresented:titlevisibility:actions:message:))
