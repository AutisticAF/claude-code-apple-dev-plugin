---
name: swiftui-observable
description: SwiftUI @Observable macro patterns including migration from ObservableObject, @Bindable, fine-grained observation, and @Observable in SwiftUI views. Use when managing state with the Observation framework.
---

# SwiftUI @Observable Patterns

## When This Skill Activates

- User creates or refactors a SwiftUI model/view-model class
- User asks about `@Observable`, `@Bindable`, or the Observation framework
- User migrates from `ObservableObject` / `@Published` / `@ObservedObject`
- User asks about fine-grained view updates or reducing unnecessary redraws
- User combines `@Observable` with `@Environment` or SwiftData `@Model`
- User asks which property wrapper to use for state management

## API Availability

| Symbol | Minimum OS |
|---|---|
| `@Observable` | iOS 17, macOS 14, watchOS 10, tvOS 17, visionOS 1 |
| `@Bindable` | iOS 17, macOS 14, watchOS 10, tvOS 17, visionOS 1 |
| `@ObservationIgnored` | iOS 17, macOS 14, watchOS 10, tvOS 17, visionOS 1 |
| `ObservableObject` | iOS 13, macOS 10.15, watchOS 6, tvOS 13 |
| `@StateObject` | iOS 14, macOS 11, watchOS 7, tvOS 14 |

## Decision Tree

```
Need to store state?
├── Value type (struct/enum) ─────────────> @State
├── Reference type (class), iOS 17+ ──────> @Observable
└── Reference type (class), iOS 13-16 ────> ObservableObject + @Published
```

Prefer `@Observable` for new class-based models on iOS 17+. Use `ObservableObject` only for iOS 13-16 support. Use `@State` for value-type state owned by a view.

## @Observable Basics

```swift
import Observation

@Observable
class UserProfile {
    var name: String = ""
    var email: String = ""
    var loginCount: Int = 0

    // No @Published needed - all stored properties are tracked automatically
}
```

The macro synthesizes `Observable` conformance, property accessors for read/mutation tracking, and an internal `ObservationRegistrar`. No Combine import or per-property annotations needed.

## Using @Observable in Views

### Owned by the View (@State)

```swift
struct ProfileView: View {
    @State private var profile = UserProfile()

    var body: some View {
        VStack {
            Text(profile.name)
            Text(profile.email)
        }
    }
}
```

### Passed as a Parameter (No Wrapper Needed)

```swift
struct ProfileDetail: View {
    var profile: UserProfile  // plain property

    var body: some View {
        Text(profile.name)
    }
}
```

### Via @Environment

```swift
struct ContentView: View {
    @State private var profile = UserProfile()

    var body: some View {
        ChildView()
            .environment(profile)
    }
}

struct ChildView: View {
    @Environment(UserProfile.self) private var profile

    var body: some View {
        Text(profile.name)
    }
}
```

## @Bindable for Two-Way Bindings

`@Bindable` creates `Binding` values from `@Observable` properties.

```swift
// On a passed-in observable:
struct ProfileEditor: View {
    @Bindable var profile: UserProfile

    var body: some View {
        TextField("Name", text: $profile.name)
        TextField("Email", text: $profile.email)
    }
}

// When stored in @State, bindings come from @State's projected value:
struct OwnerView: View {
    @State private var profile = UserProfile()
    var body: some View {
        TextField("Name", text: $profile.name)
    }
}

// For @Environment objects, create a local @Bindable:
struct EnvEditor: View {
    @Environment(UserProfile.self) private var profile
    var body: some View {
        @Bindable var profile = profile
        TextField("Name", text: $profile.name)
    }
}
```

## Fine-Grained Observation

Only properties **read inside `body`** cause the view to re-evaluate. This is automatic.

```swift
@Observable
class Settings {
    var theme: String = "light"
    var fontSize: Int = 14
    var lastSync: Date = .now  // updated frequently
}

struct ThemeLabel: View {
    var settings: Settings

    var body: some View {
        // This view re-renders only when `theme` changes.
        // Changes to fontSize or lastSync do NOT trigger an update.
        Text(settings.theme)
    }
}
```

Unlike `ObservableObject` where **any** `@Published` change re-evaluates every subscriber.

## @ObservationIgnored

Exclude properties from observation tracking. Changes to ignored properties never trigger view updates.

```swift
@Observable
class DataStore {
    var items: [Item] = []

    @ObservationIgnored var internalCache: [String: Item] = [:]
    @ObservationIgnored let logger = Logger(subsystem: "app", category: "store")
}
```

## Computed Properties

Computed properties depending on tracked stored properties are automatically observed. A view reading `totalPrice` re-renders when `items` changes.

```swift
@Observable
class Cart {
    var items: [CartItem] = []
    var totalPrice: Decimal { items.reduce(0) { $0 + $1.price } }
    var isEmpty: Bool { items.isEmpty }
}
```

## Migration from ObservableObject

### Step-by-Step

| # | Action |
|---|--------|
| 1 | Replace `class MyModel: ObservableObject` with `@Observable class MyModel` |
| 2 | Remove every `@Published` annotation |
| 3 | Remove `import Combine` if no other Combine usage remains |
| 4 | Replace `@StateObject` with `@State` |
| 5 | Replace `@ObservedObject` with a plain property (or `@Bindable` if you need `$` bindings) |
| 6 | Replace `@EnvironmentObject` with `@Environment(MyModel.self)` |
| 7 | Replace `.environmentObject(model)` with `.environment(model)` |

### Before (ObservableObject)

```swift
import Combine

class CounterModel: ObservableObject {
    @Published var count = 0
}

struct CounterView: View {
    @StateObject private var model = CounterModel()

    var body: some View {
        Button("Count: \(model.count)") {
            model.count += 1
        }
    }
}
```

### After (@Observable)

```swift
@Observable
class CounterModel {
    var count = 0
}

struct CounterView: View {
    @State private var model = CounterModel()

    var body: some View {
        Button("Count: \(model.count)") {
            model.count += 1
        }
    }
}
```

## @Observable with @Environment (Custom Key)

```swift
@Observable class AppSettings {
    var accentColor: String = "blue"
}

struct AppSettingsKey: EnvironmentKey {
    static let defaultValue = AppSettings()
}

extension EnvironmentValues {
    var appSettings: AppSettings {
        get { self[AppSettingsKey.self] }
        set { self[AppSettingsKey.self] = newValue }
    }
}

// Injection:  .environment(\.appSettings, AppSettings())
// Reading:    @Environment(\.appSettings) private var settings
```

## @Observable with SwiftData @Model

`@Model` already synthesizes `Observable` conformance. Do **not** stack `@Observable` on `@Model`.

```swift
import SwiftData

@Model class Task {
    var title: String
    var isComplete: Bool
    init(title: String, isComplete: Bool = false) {
        self.title = title
        self.isComplete = isComplete
    }
}

// Use like any @Observable object -- @Bindable for bindings:
struct TaskRow: View {
    @Bindable var task: Task
    var body: some View { Toggle(task.title, isOn: $task.isComplete) }
}
```

## Threading: @MainActor

Annotate observable classes that drive UI with `@MainActor` to guarantee mutations happen on the main thread.

```swift
@MainActor @Observable
class ViewModel {
    var results: [String] = []
    var isLoading = false

    func fetch() async {
        isLoading = true
        results = await NetworkService.fetchItems()
        isLoading = false
    }
}
```

Without `@MainActor`, mutating tracked properties from a background task risks data races.

## Backward Compatibility

| Scenario | Recommendation |
|---|---|
| iOS 17+ only | Use `@Observable` exclusively |
| iOS 16 + iOS 17 | Use `ObservableObject`; migrate when you drop iOS 16 |
| Mixed codebase | `@Observable` class can also conform to `ObservableObject` for bridging |

```swift
// Temporary bridge: older views use @ObservedObject, newer views use plain property
@Observable class SharedModel: ObservableObject {
    var value: Int = 0
}
```

## Patterns

### ✅ Good

```swift
@State private var viewModel = MyViewModel()              // owns the object
var viewModel: MyViewModel                                 // passed-in, no wrapper
@Bindable var viewModel: MyViewModel                       // need $ bindings
@Environment(MyViewModel.self) private var viewModel       // type-based env
@MainActor @Observable class MyViewModel { ... }           // safe threading
@ObservationIgnored var cache: NSCache<NSString, UIImage> = .init()
```

### ❌ Bad

```swift
// ❌ @StateObject with @Observable -- requires ObservableObject
@StateObject private var viewModel = MyViewModel()

// ❌ @ObservedObject with @Observable -- wrong protocol
@ObservedObject var viewModel: MyViewModel

// ❌ Stacking @Observable on @Model -- already observable
@Observable @Model class Task { ... }

// ❌ @Published inside @Observable -- not needed, causes warnings
@Observable class Bad { @Published var name = "" }

// ❌ Mutating off main thread without @MainActor
Task.detached { viewModel.results = newData }

// ❌ .environmentObject() with @Observable -- use .environment()
.environmentObject(viewModel)
```
