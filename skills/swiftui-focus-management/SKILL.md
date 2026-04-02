---
name: swiftui-focus-management
description: SwiftUI focus and keyboard management including @FocusState, FocusedValue, keyboard toolbar, submit triggers, and text field navigation. Use when managing focus, keyboard behavior, or text input flow.
---

> **First step:** Tell the user: "swiftui-focus-management skill loaded."

# SwiftUI Focus Management

Patterns for controlling focus, keyboard behavior, and text input navigation in SwiftUI. Covers single-field focus, multi-field form navigation, keyboard toolbars, submit handling, and cross-platform focus APIs.

## When This Skill Activates

Use this skill when the user:
- Needs to control which text field has focus
- Wants to move focus between fields (tab-through forms)
- Asks about dismissing the keyboard programmatically
- Needs a keyboard toolbar with Done/Next buttons
- Wants to customize the return key label or behavior
- Asks about @FocusState, FocusedValue, or FocusedBinding
- Needs focus management for macOS menu commands
- Wants to make non-text views focusable
- Asks about default focus or focus sections (tvOS)

## Decision Tree

```
What focus behavior do you need?
|
+- Single text field focus toggle
|  +- @FocusState with Bool -> Single Field section
|
+- Navigate between multiple fields
|  +- @FocusState with enum -> Multi-Field section
|
+- Keyboard toolbar (Done, Next, Previous)
|  +- .toolbar { ToolbarItemGroup(placement: .keyboard) }
|
+- Customize return key action
|  +- .onSubmit + .submitLabel -> Submit section
|
+- macOS menu bar commands reading focused data
|  +- FocusedValue / FocusedBinding -> Menu Commands section
|
+- Make non-text view focusable
|  +- .focusable() -> Non-Text Focus section
|
+- Set initial focus on appear
|  +- .defaultFocus() -> Default Focus section
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `@FocusState` | iOS 15 | Bool or Hashable enum binding |
| `.focused(_:)` | iOS 15 | Bool-based focus binding |
| `.focused(_:equals:)` | iOS 15 | Enum-based focus binding |
| `.onSubmit` | iOS 15 | Return key action |
| `.submitLabel(_:)` | iOS 15 | .done, .next, .search, etc. |
| `FocusedValue` | iOS 14 | Read-only value from focused view |
| `FocusedBinding` | iOS 15 | Writable binding from focused view |
| `.focusable()` | iOS 14 | Make non-text views focusable |
| `.focusSection()` | tvOS 15 | Group views for spatial focus |
| `.defaultFocus(_:_:)` | iOS 16 / macOS 13 | Set initial focus target |
| `.prefersDefaultFocus(_:in:)` | tvOS 15 / macOS 13 | Preferred focus in scope |
| `@FocusedObject` | iOS 16 | ObservableObject from focused view |
| `.focusScope(_:)` | tvOS 15 / macOS 13 | Define a focus scope namespace |

## Single Field: @FocusState with Bool

Use a `Bool` binding when you have one field to toggle focus on and off.

```swift
struct SearchBar: View {
    @State private var query = ""
    @FocusState private var isSearchFocused: Bool

    var body: some View {
        VStack {
            TextField("Search...", text: $query)
                .focused($isSearchFocused)
                .onSubmit { performSearch() }

            Button("Cancel") {
                isSearchFocused = false // dismisses keyboard
            }
        }
        .onAppear {
            isSearchFocused = true // auto-focus on appear
        }
    }
}
```

## Multi-Field Forms: @FocusState with Enum

Use a `Hashable` enum when navigating between multiple fields.

```swift
struct SignUpForm: View {
    enum Field: Hashable {
        case username, email, password
    }

    @State private var username = ""
    @State private var email = ""
    @State private var password = ""
    @FocusState private var focusedField: Field?

    var body: some View {
        Form {
            TextField("Username", text: $username)
                .focused($focusedField, equals: .username)
                .submitLabel(.next)
                .onSubmit { focusedField = .email }

            TextField("Email", text: $email)
                .focused($focusedField, equals: .email)
                .submitLabel(.next)
                .onSubmit { focusedField = .password }

            SecureField("Password", text: $password)
                .focused($focusedField, equals: .password)
                .submitLabel(.done)
                .onSubmit { submit() }
        }
        .toolbar {
            ToolbarItemGroup(placement: .keyboard) {
                Button("Previous") { moveFocus(-.1) }
                Button("Next") { moveFocus(+.1) }
                Spacer()
                Button("Done") { focusedField = nil }
            }
        }
    }

    private func moveFocus(_ direction: Double) {
        let allFields: [Field] = [.username, .email, .password]
        guard let current = focusedField,
              let index = allFields.firstIndex(of: current) else { return }
        let next = direction > 0 ? index + 1 : index - 1
        focusedField = allFields.indices.contains(next) ? allFields[next] : nil
    }
}
```

## .onSubmit and .submitLabel

Control the return key label and action.

```swift
TextField("Search", text: $query)
    .submitLabel(.search)     // shows "Search" on return key
    .onSubmit { performSearch() }

// Available submit labels:
// .done, .go, .join, .next, .return, .route, .search, .send, .continue
```

Nested `.onSubmit` propagates up the view hierarchy:

```swift
Form {
    TextField("Name", text: $name)
    TextField("Bio", text: $bio)
}
.onSubmit { saveProfile() } // called for any field in the form
```

## Keyboard Toolbar

Add buttons above the keyboard using `ToolbarItemGroup(placement: .keyboard)`.

```swift
struct NotesEditor: View {
    @State private var text = ""
    @FocusState private var isFocused: Bool

    var body: some View {
        TextEditor(text: $text)
            .focused($isFocused)
            .toolbar {
                ToolbarItemGroup(placement: .keyboard) {
                    Button(action: insertBullet) {
                        Image(systemName: "list.bullet")
                    }
                    Spacer()
                    Button("Done") { isFocused = false }
                }
            }
    }
}
```

## FocusedValue and FocusedBinding (macOS Menu Commands)

Expose data from the focused view to menu bar commands.

```swift
// 1. Define the focused value key
struct FocusedDocumentKey: FocusedValueKey {
    typealias Value = Binding<String>
}

extension FocusedValues {
    var documentText: Binding<String>? {
        get { self[FocusedDocumentKey.self] }
        set { self[FocusedDocumentKey.self] = newValue }
    }
}

// 2. Publish from the focused view
struct EditorView: View {
    @State private var text = ""

    var body: some View {
        TextEditor(text: $text)
            .focusedValue(\.documentText, $text)
    }
}

// 3. Read in menu commands
struct AppCommands: Commands {
    @FocusedBinding(\.documentText) var text

    var body: some Commands {
        CommandMenu("Format") {
            Button("Uppercase") {
                text = text?.uppercased() ?? ""
            }
            .disabled(text == nil)
        }
    }
}
```

## Non-Text View Focus with .focusable()

Make buttons, images, or custom views participate in focus navigation.

```swift
struct FocusableCard: View {
    @FocusState private var isFocused: Bool

    var body: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(isFocused ? Color.blue : Color.gray)
            .frame(width: 200, height: 120)
            .focusable()
            .focused($isFocused)
            .onKeyPress(.return) {
                handleSelection()
                return .handled
            }
    }
}
```

## Focus Sections (tvOS / Spatial Grouping)

Group views into a single focusable region so the focus engine treats them as a unit.

```swift
HStack {
    VStack {
        Button("A") {}
        Button("B") {}
    }
    .focusSection() // focus moves into/out of this group as a unit

    VStack {
        Button("C") {}
        Button("D") {}
    }
    .focusSection()
}
```

## Default Focus

Set which view receives focus when a scene or scope first appears.

```swift
struct SettingsPanel: View {
    @FocusState private var focusedField: Field?
    @Namespace private var settingsScope

    var body: some View {
        Form {
            TextField("Name", text: $name)
                .focused($focusedField, equals: .name)

            TextField("Email", text: $email)
                .focused($focusedField, equals: .email)
        }
        .focusScope(settingsScope)
        .defaultFocus($focusedField, .name, priority: .userInitiated)
    }
}
```

For tvOS or complex layouts, use `.prefersDefaultFocus` within a scope:

```swift
VStack {
    Button("Primary") { }
        .prefersDefaultFocus(in: scopeNamespace)
    Button("Secondary") { }
}
.focusScope(scopeNamespace)
```

## Dismissing the Keyboard

Set `@FocusState` to `nil` (enum) or `false` (Bool) to resign first responder.

```swift
// Tap-outside-to-dismiss pattern
struct DismissableForm: View {
    @FocusState private var focusedField: Field?

    var body: some View {
        Form {
            TextField("Name", text: $name)
                .focused($focusedField, equals: .name)
        }
        .onTapGesture { focusedField = nil }
        // Or use scrollDismissesKeyboard for scroll views:
        .scrollDismissesKeyboard(.interactively) // iOS 16+
    }
}
```

## Patterns

### ✅ Good Patterns

```swift
// ✅ Use Optional enum for multi-field — nil means no focus
@FocusState private var field: Field?

// ✅ Dismiss keyboard by setting focus to nil
Button("Done") { focusedField = nil }

// ✅ Chain submitLabel with onSubmit for clear field progression
TextField("Email", text: $email)
    .submitLabel(.next)
    .onSubmit { focusedField = .password }

// ✅ Use .defaultFocus to set initial field on appear
.defaultFocus($focusedField, .username, priority: .userInitiated)

// ✅ Provide keyboard toolbar for fields without a return key (number pad)
.toolbar {
    ToolbarItemGroup(placement: .keyboard) {
        Spacer()
        Button("Done") { focusedField = nil }
    }
}
```

### ❌ Bad Patterns

```swift
// ❌ Don't use DispatchQueue to set focus — use @FocusState directly
DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
    UIApplication.shared.sendAction(#selector(UIResponder.becomeFirstResponder),
                                     to: nil, from: nil, for: nil)
}

// ❌ Don't use UIKit responder chain to dismiss keyboard in SwiftUI
UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder),
                                 to: nil, from: nil, for: nil)

// ❌ Don't forget that @FocusState enum must be Optional
@FocusState private var field: Field // Compiler error — must be Field?

// ❌ Don't use .focused() without a matching @FocusState property
TextField("Name", text: $name)
    .focused($someStateVar, equals: .name) // $someStateVar must be @FocusState

// ❌ Don't set @FocusState in init or outside the view lifecycle
init() {
    _focusedField = .init(wrappedValue: .username) // won't work as expected
}
```
