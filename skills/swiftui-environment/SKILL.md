---
name: swiftui-environment
description: SwiftUI Environment and Preferences system including custom EnvironmentKey, EnvironmentValues, @Environment, preference keys, and the preference-environment data flow. Use when passing data through the view hierarchy.
---

# SwiftUI Environment & Preferences

Data propagation through the SwiftUI view hierarchy. Environment flows data downward from parent to child. Preferences flow data upward from child to parent.

## When This Skill Activates

Use this skill when the user:
- Needs to pass data through the view hierarchy without explicit init parameters
- Wants to create a custom `EnvironmentKey` or extend `EnvironmentValues`
- Asks about `@Environment`, `.environment()`, or environment injection
- Wants to inject services or repositories for dependency injection
- Needs child views to communicate measurements or values upward (preferences)
- Asks about `PreferenceKey`, `.onPreferenceChange`, or `.overlayPreferenceValue`
- Is deciding between Environment, Preferences, @Binding, or init parameters
- Wants to replace `GeometryReader` with preference-based measurement
- Asks about the `@Entry` macro for simplified environment keys (iOS 18+)

## Decision Tree

```
How should data flow through the hierarchy?
|
+- Parent provides value to all descendants (downward)
|  |
|  +- Value is view-scoped (theme, locale, services)
|  |  +- Use Environment
|  |
|  +- Value is two-way (parent and child both read/write)
|     +- Use @Binding
|
+- Child communicates a value to ancestors (upward)
|  |
|  +- Child reports a measurement (size, position, anchor)
|  |  +- Use PreferenceKey
|  |
|  +- Ancestor renders an overlay/background based on child data
|     +- Use .overlayPreferenceValue / .backgroundPreferenceValue
|
+- Value only needed by immediate child
|  +- Pass via init parameter
|
+- Value needed by deeply nested descendants
   +- Use Environment (avoids prop drilling)
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `@Environment(\.key)` | iOS 13 | Read environment values |
| `.environment(\.key, value)` | iOS 13 | Inject environment values |
| `EnvironmentKey` protocol | iOS 13 | Custom environment keys |
| `PreferenceKey` protocol | iOS 13 | Child-to-parent data flow |
| `.onPreferenceChange` | iOS 13 | React to preference updates |
| `.overlayPreferenceValue` | iOS 13 | Overlay using preference data |
| `.backgroundPreferenceValue` | iOS 13 | Background using preference data |
| `.transformEnvironment` | iOS 13 | Mutate environment in-place |
| `@Environment(SomeType.self)` | iOS 17 | Observable environment objects |
| `@Entry` macro | iOS 18 | Simplified EnvironmentKey definition |

## Built-in Environment Values

| Key Path | Type | Purpose |
|----------|------|---------|
| `\.colorScheme` | `ColorScheme` | Light or dark appearance |
| `\.dismiss` | `DismissAction` | Dismiss current presentation |
| `\.openURL` | `OpenURLAction` | Open a URL |
| `\.locale` | `Locale` | Current locale |
| `\.horizontalSizeClass` | `UserInterfaceSizeClass?` | Compact or regular width |
| `\.verticalSizeClass` | `UserInterfaceSizeClass?` | Compact or regular height |
| `\.isEnabled` | `Bool` | Whether the view is enabled |
| `\.editMode` | `Binding<EditMode>?` | List edit mode |
| `\.managedObjectContext` | `NSManagedObjectContext` | Core Data context |
| `\.openWindow` | `OpenWindowAction` | Open a new window (macOS/iPadOS) |
| `\.requestReview` | `RequestReviewAction` | Request App Store review |

## Custom EnvironmentKey (iOS 13+)

### Step 1: Define the Key

```swift
struct ThemeKey: EnvironmentKey {
    static let defaultValue: Theme = .standard
}
```

### Step 2: Extend EnvironmentValues

```swift
extension EnvironmentValues {
    var theme: Theme {
        get { self[ThemeKey.self] }
        set { self[ThemeKey.self] = newValue }
    }
}
```

### Step 3: Read with @Environment

```swift
struct ProfileView: View {
    @Environment(\.theme) private var theme

    var body: some View {
        Text("Hello")
            .foregroundStyle(theme.primaryColor)
    }
}
```

### Step 4: Inject with .environment

```swift
ContentView()
    .environment(\.theme, .dark)
```

## @Entry Macro (iOS 18+)

The `@Entry` macro eliminates the boilerplate of defining a separate key struct:

```swift
extension EnvironmentValues {
    @Entry var theme: Theme = .standard
}
```

This single line replaces the `EnvironmentKey` struct and the computed property. Reading and injection use the same `@Environment(\.theme)` and `.environment(\.theme, value)` syntax.

## Environment for Dependency Injection

```swift
protocol NetworkService: Sendable {
    func fetch(_ url: URL) async throws -> Data
}

extension EnvironmentValues {
    @Entry var networkService: any NetworkService = URLSessionNetworkService()
}

struct UserListView: View {
    @Environment(\.networkService) private var network

    var body: some View {
        List { /* ... */ }
            .task { let data = try? await network.fetch(usersURL) }
    }
}

// Swap for previews or tests
#Preview {
    UserListView()
        .environment(\.networkService, MockNetworkService())
}
```

## Transform Environment

Modify an inherited environment value in-place rather than replacing it:

```swift
MyView()
    .transformEnvironment(\.theme) { theme in
        theme.cornerRadius = 16
    }
```

## Preference Keys

### Define a PreferenceKey

```swift
struct WidthPreferenceKey: PreferenceKey {
    static var defaultValue: CGFloat = 0
    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = max(value, nextValue())
    }
}
```

Common `reduce` strategies: `max(value, nextValue())` for largest wins, `value + nextValue()` for totals, `value = nextValue()` for last writer wins, `value.append(contentsOf: nextValue())` for collections.

### Report and Read Preferences

```swift
// Child reports its width via a background GeometryReader
Text("Hello")
    .background(
        GeometryReader { proxy in
            Color.clear
                .preference(key: WidthPreferenceKey.self, value: proxy.size.width)
        }
    )

// Ancestor reads the aggregated preference
.onPreferenceChange(WidthPreferenceKey.self) { width in
    maxChildWidth = width
}
```

### Overlay and Background with Preferences

Use `.anchorPreference` to report anchors, then `.overlayPreferenceValue` to render decorations:

```swift
struct BoundsPreferenceKey: PreferenceKey {
    static var defaultValue: Anchor<CGRect>? = nil
    static func reduce(value: inout Anchor<CGRect>?, nextValue: () -> Anchor<CGRect>?) {
        value = value ?? nextValue()
    }
}

// Child reports its bounds
Text("Important")
    .anchorPreference(key: BoundsPreferenceKey.self, value: .bounds) { $0 }

// Ancestor draws overlay using the anchor
.overlayPreferenceValue(BoundsPreferenceKey.self) { anchor in
    if let anchor {
        GeometryReader { proxy in
            let rect = proxy[anchor]
            RoundedRectangle(cornerRadius: 8)
                .stroke(.blue, lineWidth: 2)
                .frame(width: rect.width, height: rect.height)
                .offset(x: rect.minX, y: rect.minY)
        }
    }
}
```

## Data Flow Direction

```
   .environment(\.theme, .dark)
           |
           v  (Environment flows DOWN)
       ParentView
        /      \
   ChildA     ChildB
      |
   GrandchildA
      |
      ^  (Preferences flow UP)
   .preference(key:value:)
```

- **Environment**: Set by an ancestor, readable by any descendant. Changes trigger re-renders only in views that read the key.
- **Preferences**: Set by a descendant, readable by any ancestor. The `reduce` function merges values from sibling subtrees.

## GeometryReader vs Preferences for Measuring

| Approach | Pros | Cons |
|----------|------|------|
| `GeometryReader` | Simple, immediate | Consumes all proposed space; disrupts layout |
| Preference + background `GeometryReader` | Non-intrusive; child drives own size | More boilerplate |

Prefer the preference approach when measuring without affecting layout.

## Patterns

### Pass shared configuration through Environment

```swift
// ✅ Good: inject once, available everywhere
ContentView()
    .environment(\.theme, appTheme)
    .environment(\.networkService, liveNetwork)
```

```swift
// ❌ Bad: drilling a parameter through every init
ContentView(theme: appTheme)
    // -> SubView(theme: appTheme)
    //    -> DetailView(theme: appTheme)
    //       -> CardView(theme: appTheme)
```

### Provide a sensible default value

```swift
// ✅ Good: default value means views work without explicit injection
struct AccentStyleKey: EnvironmentKey {
    static let defaultValue: Color = .accentColor
}
```

```swift
// ❌ Bad: force-unwrapping or crashing when no value is injected
struct AccentStyleKey: EnvironmentKey {
    static let defaultValue: Color? = nil  // then force-unwrapping later
}
```

### Keep preference reduce functions pure

```swift
// ✅ Good: pure function, predictable behavior
static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
    value = max(value, nextValue())
}
```

```swift
// ❌ Bad: side effects in reduce
static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
    let next = nextValue()
    print("Reducing: \(value) with \(next)")  // side effect
    value = next
}
```

### Use @Entry for new projects targeting iOS 18+

```swift
// ✅ Good: concise, less boilerplate (iOS 18+)
extension EnvironmentValues {
    @Entry var accentStyle: Color = .blue
}
```

```swift
// ❌ Unnecessary when targeting iOS 18+: verbose three-part definition
struct AccentStyleKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}
extension EnvironmentValues {
    var accentStyle: Color {
        get { self[AccentStyleKey.self] }
        set { self[AccentStyleKey.self] = newValue }
    }
}
```

### Read environment in body, not init

```swift
// ✅ Good: @Environment is populated when body is called
@Environment(\.theme) private var theme
var body: some View { RoundedRectangle(cornerRadius: theme.cornerRadius) }
```

```swift
// ❌ Bad: @Environment values are NOT available during init
init() { cornerRadius = 12 /* cannot read theme here */ }
```
