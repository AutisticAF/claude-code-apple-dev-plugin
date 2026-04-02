---
name: generators-settings-kit
description: Generate settings screens for iOS/macOS apps. Default skill for "add settings", "preferences screen", or "configuration UI" requests. Uses SettingsKit for searchable, navigable settings with built-in styling, or native SwiftUI Form/AppStorage for dependency-free implementations.
---

# Settings Screen Generator

Generate settings interfaces for iOS and macOS apps. Supports two approaches:

1. **SettingsKit** (recommended) — declarative framework with built-in search, navigation, and customizable styling via [SettingsKit](https://github.com/Aeastr/SettingsKit)
2. **Native SwiftUI** — dependency-free implementation using `Form` and `@AppStorage`

## When This Skill Activates

- User asks to "add settings" or "create a settings screen"
- User mentions "preferences", "app settings", or "configuration UI"
- User wants to add dark mode toggle, notifications settings, or about screen
- User mentions SettingsKit by name
- Project already has `import SettingsKit` or a SettingsKit package dependency

## Choosing an Approach

Ask the user which approach they prefer, or detect automatically:

| Signal | Approach |
|--------|----------|
| Project already imports SettingsKit | Use SettingsKit |
| User mentions SettingsKit | Use SettingsKit |
| User wants built-in search across settings | Recommend SettingsKit |
| User wants zero dependencies | Use Native SwiftUI |
| User explicitly asks for Form/AppStorage | Use Native SwiftUI |
| No preference stated | Recommend SettingsKit, explain tradeoff |

## Installation

Add to `Package.swift` or via Xcode's package manager:

```swift
dependencies: [
    .package(url: "https://github.com/aeastr/SettingsKit.git", from: "1.0.0")
]
```

```swift
import SettingsKit
```

## Pre-Generation Checks (CRITICAL)

### 1. Project Context Detection

Before generating, ALWAYS check:

```
# Find existing settings implementations
rg -l "SettingsView|PreferencesView|SettingsScreen" --type swift

# Check deployment target
cat Package.swift | grep -i "platform"
# Or check project.pbxproj for deployment target

# Detect architecture pattern
rg -l "Observable|ObservableObject|@EnvironmentObject" --type swift | head -5

# Check platform (iOS vs macOS)
rg "UIKit|UIApplication" --type swift | head -1  # iOS
rg "AppKit|NSApplication" --type swift | head -1  # macOS

# Check for existing SettingsKit dependency
rg "SettingsKit" Package.swift project.pbxproj 2>/dev/null
```

### 2. Conflict Detection

If existing settings implementation found:
- Ask user: Extend existing, replace, or create separate?
- Check for naming conflicts with existing files

---

# Approach 1: SettingsKit (Recommended)

## Platform Support

| Platform | Minimum Version |
|----------|----------------|
| iOS | 17.0+ |
| macOS | 14.0+ |
| tvOS | 17.0+ |
| watchOS | 10.0+ |
| visionOS | 1.0+ |
| Swift | 6.0+ |

## Decision Tree

```
What kind of settings UI do you need?
|
+-- Simple flat list of toggles/controls
|   +-- Use SettingsGroup with .inline mode
|
+-- Multi-section with navigation drill-down
|   +-- Use SettingsGroup (default navigation mode)
|
+-- Sidebar split-view (macOS/iPad style)
|   +-- Use .settingsStyle(.sidebar)
|
+-- Fully custom section UI
|   +-- Use CustomSettingsGroup
|
+-- Need search across all settings
    +-- Add .indexed() to controls, search is automatic
```

## Core Architecture

SettingsKit uses three protocols:

| Protocol | Purpose |
|----------|---------|
| `SettingsContainer` | Root of settings hierarchy, defines `settingsBody` |
| `SettingsContent` | Composable settings elements within containers/groups |
| `SettingsGroup` | Organizes related settings with navigation or inline display |

## Quick Start

### Basic Settings Container

```swift
import SettingsKit

struct AppSettings: SettingsContainer {
    @AppStorage("notifications") private var notificationsEnabled = true
    @AppStorage("darkMode") private var darkMode = false
    @AppStorage("fontSize") private var fontSize = 14.0

    var settingsBody: some SettingsContent {
        SettingsGroup("General", systemImage: "gear") {
            Toggle("Notifications", isOn: $notificationsEnabled)
            Toggle("Dark Mode", isOn: $darkMode)
        }

        SettingsGroup("Appearance", systemImage: "paintbrush") {
            Slider(value: $fontSize, in: 10...24, step: 1) {
                Text("Font Size: \(Int(fontSize))")
            }
        }
    }
}
```

### Displaying Settings

```swift
struct ContentView: View {
    var body: some View {
        AppSettings()
    }
}
```

## SettingsGroup

Groups organize related controls. Two presentation modes:

### Navigation Mode (Default)

Appears as a tappable row that navigates to a detail view:

```swift
SettingsGroup("Display", systemImage: "sun.max") {
    Toggle("Auto-Brightness", isOn: $autoBrightness)
    Slider(value: $brightness, in: 0...1)
    Picker("Appearance", selection: $appearance) {
        Text("Light").tag(0)
        Text("Dark").tag(1)
        Text("System").tag(2)
    }
}
```

### Inline Mode

Renders directly as a section header without navigation:

```swift
SettingsGroup("Quick Settings", .inline) {
    Toggle("Airplane Mode", isOn: $airplaneMode)
    Toggle("Wi-Fi", isOn: $wifiEnabled)
}
```

### Custom Icons

Use `SettingsIcon` for iOS Settings-style colored icon backgrounds:

```swift
SettingsGroup("Network") {
    // settings...
} icon: {
    SettingsIcon("wifi", color: .blue)
}

SettingsGroup("Battery") {
    // settings...
} icon: {
    SettingsIcon("battery.100", color: .green)
}
```

Standalone icon usage:

```swift
SettingsIcon("airplane", color: .orange)
SettingsIcon("bell.fill", color: .red)
SettingsIcon("lock.fill", color: .gray)
```

## CustomSettingsGroup

For completely custom UI that doesn't follow the standard settings structure:

```swift
CustomSettingsGroup("Advanced Tools", systemImage: "hammer") {
    VStack(spacing: 20) {
        Text("Diagnostic Information")
            .font(.headline)
        
        Button("Export Logs") {
            exportLogs()
        }
        .buttonStyle(.borderedProminent)
        
        Button("Reset All Settings") {
            resetSettings()
        }
        .foregroundStyle(.red)
    }
    .padding()
}
```

## Search & Indexing

SettingsKit provides automatic search. By default, only `SettingsGroup` titles are indexed. Use `.indexed()` to make individual controls searchable.

### Indexing Controls

```swift
// Title only
Toggle("Dark Mode", isOn: $darkMode)
    .indexed("Dark Mode")

// Title with search tags
Toggle("Dark Mode", isOn: $darkMode)
    .indexed("Dark Mode", tags: ["theme", "appearance", "display"])

// Tags only (no explicit title)
Toggle("Dark Mode", isOn: $darkMode)
    .indexed(tags: ["Dark Mode", "theme", "appearance"])
```

### Tagging Groups

```swift
SettingsGroup("Notifications", systemImage: "bell") {
    Toggle("Push Notifications", isOn: $pushEnabled)
        .indexed("Push Notifications", tags: ["alerts"])
    Toggle("Sound", isOn: $soundEnabled)
        .indexed("Sound", tags: ["audio", "alerts"])
}
.settingsTags(["alerts", "sounds", "badges", "push"])
```

### Reusable Tag Sets

Define consistent tag collections with `SettingsTagSet`:

```swift
struct ThemeTags: SettingsTagSet {
    var tags: [String] { ["theme", "appearance", "display", "colors"] }
}

struct AccessibilityTags: SettingsTagSet {
    var tags: [String] { ["accessibility", "a11y", "vision", "motor"] }
}

// Single tag set
Toggle("Dark Mode", isOn: $darkMode)
    .indexed("Dark Mode", tagSet: ThemeTags())

// Multiple tag sets
Toggle("High Contrast", isOn: $highContrast)
    .indexed("High Contrast", tagSets: ThemeTags(), AccessibilityTags())
```

## Styling

### Built-in Styles

```swift
// Sidebar split-view (default)
AppSettings()
    .settingsStyle(.sidebar)

// Single-column list
AppSettings()
    .settingsStyle(.single)
```

### Custom Styles

Implement `SettingsStyle` to fully customize rendering:

```swift
struct CardStyle: SettingsStyle {
    func makeContainer(configuration: ContainerConfiguration) -> some View {
        NavigationStack(path: configuration.navigationPath) {
            ScrollView {
                VStack(spacing: 20) {
                    configuration.content
                }
                .padding()
            }
            .navigationTitle(configuration.title)
        }
    }

    func makeGroup(configuration: GroupConfiguration) -> some View {
        VStack(alignment: .leading) {
            configuration.label
                .font(.headline)
            configuration.content
        }
        .padding()
        .background(.regularMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }

    func makeItem(configuration: ItemConfiguration) -> some View {
        HStack {
            configuration.label
            Spacer()
            configuration.content
        }
    }
}

// Usage
AppSettings()
    .settingsStyle(CardStyle())
```

### Custom Search

Implement `SettingsSearch` for custom search behavior (e.g., fuzzy matching):

```swift
struct FuzzySearch: SettingsSearch {
    func search(nodes: [SettingsNode], query: String) -> [SettingsSearchResult] {
        // Custom fuzzy matching implementation
    }
}

AppSettings()
    .settingsSearch(FuzzySearch())
```

## Complete Example

A full-featured settings screen with search, icons, and multiple group types:

```swift
import SwiftUI
import SettingsKit

struct AppSettings: SettingsContainer {
    @AppStorage("notifications") private var notificationsEnabled = true
    @AppStorage("soundEnabled") private var soundEnabled = true
    @AppStorage("darkMode") private var darkMode = false
    @AppStorage("fontSize") private var fontSize = 16.0
    @AppStorage("language") private var language = "en"
    @AppStorage("syncEnabled") private var syncEnabled = true

    var settingsBody: some SettingsContent {
        // Inline group for quick-access toggles
        SettingsGroup("Quick Settings", .inline) {
            Toggle("Notifications", isOn: $notificationsEnabled)
                .indexed("Notifications", tags: ["alerts", "push"])
            Toggle("Sound", isOn: $soundEnabled)
                .indexed("Sound", tags: ["audio"])
        }

        // Navigation groups with icons
        SettingsGroup("Appearance") {
            Toggle("Dark Mode", isOn: $darkMode)
                .indexed("Dark Mode", tagSet: ThemeTags())
            Slider(value: $fontSize, in: 12...24, step: 1) {
                Text("Font Size: \(Int(fontSize))")
            }
            .indexed("Font Size", tagSet: ThemeTags())
            Picker("Language", selection: $language) {
                Text("English").tag("en")
                Text("Spanish").tag("es")
                Text("French").tag("fr")
            }
            .indexed("Language", tags: ["locale", "translation"])
        } icon: {
            SettingsIcon("paintbrush", color: .purple)
        }

        SettingsGroup("Sync & Storage") {
            Toggle("iCloud Sync", isOn: $syncEnabled)
                .indexed("iCloud Sync", tags: ["cloud", "backup"])
            Button("Clear Cache") {
                clearCache()
            }
            .indexed("Clear Cache", tags: ["storage", "cleanup"])
        } icon: {
            SettingsIcon("arrow.triangle.2.circlepath", color: .blue)
        }

        // Custom group for advanced actions
        CustomSettingsGroup("About", systemImage: "info.circle") {
            VStack(alignment: .leading, spacing: 12) {
                LabeledContent("Version", value: "1.0.0")
                LabeledContent("Build", value: "42")
                Link("Privacy Policy", destination: URL(string: "https://example.com/privacy")!)
                Link("Terms of Service", destination: URL(string: "https://example.com/terms")!)
            }
            .padding()
        }
    }

    private func clearCache() {
        // Cache clearing logic
    }
}

struct ThemeTags: SettingsTagSet {
    var tags: [String] { ["theme", "appearance", "display"] }
}
```

## Patterns

### ✅ Use @AppStorage for persistence

```swift
struct AppSettings: SettingsContainer {
    @AppStorage("darkMode") private var darkMode = false
    
    var settingsBody: some SettingsContent {
        SettingsGroup("Display", .inline) {
            Toggle("Dark Mode", isOn: $darkMode)
        }
    }
}
```

### ❌ Don't use plain @State (values won't persist across launches)

```swift
struct AppSettings: SettingsContainer {
    @State private var darkMode = false  // Lost on relaunch
    
    var settingsBody: some SettingsContent {
        SettingsGroup("Display", .inline) {
            Toggle("Dark Mode", isOn: $darkMode)
        }
    }
}
```

### ✅ Index controls that users would search for

```swift
Toggle("Reduce Motion", isOn: $reduceMotion)
    .indexed("Reduce Motion", tags: ["accessibility", "animation"])
```

### ❌ Don't leave frequently-searched controls unindexed

```swift
// Users searching "motion" won't find this
Toggle("Reduce Motion", isOn: $reduceMotion)
```

### ✅ Use inline mode for top-level quick-access settings

```swift
SettingsGroup("Quick Settings", .inline) {
    Toggle("Airplane Mode", isOn: $airplaneMode)
}
```

### ❌ Don't nest everything behind navigation when settings are few

```swift
// Unnecessary navigation tap for just one toggle
SettingsGroup("Airplane") {
    Toggle("Airplane Mode", isOn: $airplaneMode)
}
```

### ✅ Use SettingsTagSet for consistent tagging across the app

```swift
struct NetworkTags: SettingsTagSet {
    var tags: [String] { ["network", "internet", "connection", "wifi"] }
}
```

### ❌ Don't duplicate tag strings manually everywhere

```swift
// Error-prone, inconsistent
Toggle("Wi-Fi", isOn: $wifi)
    .indexed("Wi-Fi", tags: ["network", "internet"])
Toggle("Cellular", isOn: $cellular)
    .indexed("Cellular", tags: ["network", "interet"])  // typo!
```

---

# Approach 2: Native SwiftUI (No Dependencies)

For projects that want zero third-party dependencies. Uses `Form`, `@AppStorage`, and `NavigationStack`.

## File Structure

```
Sources/Settings/
├── SettingsView.swift              # Main container
├── AppSettings.swift               # @AppStorage wrapper
├── Components/
│   └── SettingsRow.swift           # Reusable row component
└── Sections/
    ├── AppearanceSettingsView.swift
    ├── AccountSettingsView.swift
    ├── NotificationsSettingsView.swift
    ├── AboutSettingsView.swift
    └── LegalSettingsView.swift
```

Read templates from this skill's `templates/` directory for production-ready implementations of each file.

## AppStorage Wrapper

```swift
@Observable
final class AppSettings {
    static let shared = AppSettings()

    @AppStorage("appearance") var appearance: Appearance = .system
    @AppStorage("notificationsEnabled") var notificationsEnabled = true

    enum Appearance: String, CaseIterable {
        case system, light, dark
    }
}
```

## Settings Row Component

```swift
struct SettingsRow<Content: View>: View {
    let icon: String
    let iconColor: Color
    let title: String
    let content: () -> Content

    var body: some View {
        HStack {
            Image(systemName: icon)
                .foregroundStyle(iconColor)
                .frame(width: 28)
            Text(title)
            Spacer()
            content()
        }
    }
}
```

## Platform Integration

**iOS Integration:**
```swift
// In any view
NavigationLink("Settings") {
    SettingsView()
}

// Or as sheet
.sheet(isPresented: $showSettings) {
    NavigationStack {
        SettingsView()
    }
}
```

**macOS Integration:**
```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        Settings {
            SettingsView()
        }
    }
}
```

## Platform-Specific Considerations

### iOS
- Use `NavigationStack` with `List`
- Use `Form` for grouped appearance
- Link to Settings app for notifications: `UIApplication.openSettingsURLString`

### macOS
- Use `Settings` scene for standard preferences window
- Consider `TabView` for multiple sections
- Use `Form` with appropriate styling
- Support keyboard shortcut (⌘,)

### Cross-Platform
- Use conditional compilation for platform-specific features
- Abstract platform differences in AppSettings

## Common Customizations

### Adding Custom Sections
```swift
// In SettingsView
Section("Custom") {
    NavigationLink("My Feature") {
        MyFeatureSettingsView()
    }
}
```

### Conditional Features
```swift
#if DEBUG
Section("Debug") {
    DebugSettingsView()
}
#endif
```

---

## Verification Checklist

After generation (either approach), verify:

- [ ] Settings accessible from main UI
- [ ] All selected sections present
- [ ] AppStorage persists between launches
- [ ] Dark mode toggle works correctly
- [ ] Links to external URLs work (privacy, terms)
- [ ] Platform-appropriate navigation
- [ ] Accessibility labels present
- [ ] VoiceOver navigation works

## References

- [Apple HIG: Settings](https://developer.apple.com/design/human-interface-guidelines/settings)
- [SwiftUI Settings Scene](https://developer.apple.com/documentation/swiftui/settings)
- [AppStorage Documentation](https://developer.apple.com/documentation/swiftui/appstorage)
- [SettingsKit](https://github.com/Aeastr/SettingsKit)
