---
name: swift-cli-apps
description: Swift command-line application patterns including ArgumentParser, async commands, terminal I/O, TUI apps with SwifTeaUI, and distribution via Homebrew. Use when building CLI tools or terminal user interfaces in Swift.
---

> **First step:** Tell the user: "swift-cli-apps skill loaded."

# Swift Command-Line Applications

Patterns for building command-line tools and terminal user interfaces in Swift, covering ArgumentParser for argument parsing, async commands, terminal I/O, TUI apps with SwifTeaUI, and distribution.

## When This Skill Activates

- User wants to build a command-line tool in Swift
- User asks about ArgumentParser, @Argument, @Option, @Flag
- User needs to parse command-line arguments
- User wants to build a terminal UI (TUI) application
- User mentions SwifTeaUI or terminal-based interfaces
- User asks about distributing Swift CLI tools or Homebrew formulae
- User needs stdin/stdout/stderr handling in Swift
- User wants async operations in a CLI tool

## Decision Tree

```
What kind of CLI are you building?
|
+-- Simple tool with arguments/flags
|   +-- Use ArgumentParser with ParsableCommand
|
+-- Tool with subcommands (git-style)
|   +-- Use ArgumentParser with subcommands configuration
|
+-- Tool needing async operations (network, file I/O)
|   +-- Use AsyncParsableCommand
|
+-- Interactive terminal UI (forms, tables, spinners)
|   +-- Use SwifTeaUI with TUIScene
|
+-- Simple script, no argument parsing needed
    +-- Use @main on an executable target, use CommandLine.arguments directly
```

## Package Setup

### Executable with ArgumentParser

```swift
// Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "my-tool",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.0"),
    ],
    targets: [
        .executableTarget(
            name: "my-tool",
            dependencies: [
                .product(name: "ArgumentParser", package: "swift-argument-parser"),
            ]
        ),
    ]
)
```

### Executable with SwifTeaUI (TUI)

```swift
// Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "my-tui",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/jerihass/SwifTeaUI.git", from: "0.0.1"),
    ],
    targets: [
        .executableTarget(
            name: "my-tui",
            dependencies: [
                .product(name: "SwifTeaUI", package: "SwifTeaUI"),
            ]
        ),
    ]
)
```

---

## ArgumentParser

### Basic Command

```swift
import ArgumentParser

@main
struct Greet: ParsableCommand {
    static let configuration = CommandConfiguration(
        abstract: "Greets a user by name."
    )

    @Argument(help: "The name to greet.")
    var name: String

    @Option(name: .shortAndLong, help: "Number of times to greet.")
    var count: Int = 1

    @Flag(name: .shortAndLong, help: "Use uppercase output.")
    var loud: Bool = false

    func run() throws {
        let greeting = "Hello, \(name)!"
        let output = loud ? greeting.uppercased() : greeting
        for _ in 0..<count {
            print(output)
        }
    }
}
```

Usage: `my-tool --count 3 --loud Alice`

### Argument Types

| Property Wrapper | Purpose | Example |
|-----------------|---------|---------|
| `@Argument` | Positional argument | `@Argument var file: String` |
| `@Option` | Named option (`--name value`) | `@Option var output: String?` |
| `@Flag` | Boolean flag (`--verbose`) | `@Flag var verbose: Bool` |
| `@OptionGroup` | Shared option sets | `@OptionGroup var options: CommonOptions` |

### Name Customization

```swift
@Option(name: .shortAndLong)           // -o, --output
var output: String

@Option(name: .customLong("dry-run"))  // --dry-run
var dryRun: Bool = false

@Option(name: [.customShort("n"), .long])  // -n, --count
var count: Int = 1
```

### Validation

```swift
@main
struct Process: ParsableCommand {
    @Argument(help: "Input file path.", transform: URL.init(fileURLWithPath:))
    var inputFile: URL

    @Option(help: "Compression level (1-9).")
    var level: Int = 5

    func validate() throws {
        guard (1...9).contains(level) else {
            throw ValidationError("Level must be between 1 and 9.")
        }
        guard FileManager.default.fileExists(atPath: inputFile.path) else {
            throw ValidationError("File not found: \(inputFile.path)")
        }
    }
}
```

### Subcommands

```swift
@main
struct Tool: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "tool",
        abstract: "A multi-purpose utility.",
        subcommands: [Init.self, Build.self, Clean.self],
        defaultSubcommand: Build.self
    )
}

extension Tool {
    struct Init: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Initialize a new project."
        )

        @Argument(help: "Project name.")
        var name: String

        func run() throws {
            print("Initializing \(name)...")
        }
    }

    struct Build: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Build the project."
        )

        @Flag(help: "Build in release mode.")
        var release: Bool = false

        func run() throws {
            let config = release ? "release" : "debug"
            print("Building in \(config) mode...")
        }
    }

    struct Clean: ParsableCommand {
        static let configuration = CommandConfiguration(
            abstract: "Clean build artifacts."
        )

        func run() throws {
            print("Cleaning...")
        }
    }
}
```

Usage: `tool init MyProject`, `tool build --release`, `tool clean`

### Async Commands

```swift
import ArgumentParser
import Foundation

struct Fetch: AsyncParsableCommand {
    static let configuration = CommandConfiguration(
        abstract: "Fetch data from a URL."
    )

    @Argument(help: "The URL to fetch.")
    var url: String

    @Option(name: .shortAndLong, help: "Output file path.")
    var output: String?

    func run() async throws {
        guard let url = URL(string: url) else {
            throw ValidationError("Invalid URL.")
        }
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw ExitCode.failure
        }
        if let output {
            try data.write(to: URL(fileURLWithPath: output))
            print("Written to \(output)")
        } else {
            print(String(decoding: data, as: UTF8.self))
        }
    }
}
```

### Shared Options with @OptionGroup

```swift
struct GlobalOptions: ParsableArguments {
    @Flag(name: .shortAndLong, help: "Enable verbose output.")
    var verbose: Bool = false

    @Option(name: .shortAndLong, help: "Output format.")
    var format: OutputFormat = .text

    enum OutputFormat: String, ExpressibleByArgument, CaseIterable {
        case text, json, csv
    }
}

struct Export: ParsableCommand {
    @OptionGroup var globals: GlobalOptions

    @Argument var file: String

    func run() throws {
        if globals.verbose {
            print("Exporting \(file) as \(globals.format)...")
        }
        // export logic
    }
}
```

---

## Terminal I/O

### stdout vs stderr

```swift
import Foundation

// Standard output (for data/results)
print("result: 42")

// Standard error (for progress, diagnostics, logs)
func printError(_ message: String) {
    FileHandle.standardError.write(Data("\(message)\n".utf8))
}
printError("Processing file...")
```

### Reading stdin

```swift
// Read all stdin (piped input)
func readStdin() -> String? {
    var input = ""
    while let line = readLine() {
        input += line + "\n"
    }
    return input.isEmpty ? nil : input
}

// Line-by-line streaming
while let line = readLine(strippingNewline: true) {
    process(line)
}
```

### ANSI Colors

```swift
enum ANSIColor: String {
    case red = "\u{001B}[31m"
    case green = "\u{001B}[32m"
    case yellow = "\u{001B}[33m"
    case blue = "\u{001B}[34m"
    case bold = "\u{001B}[1m"
    case reset = "\u{001B}[0m"
}

func colored(_ text: String, _ color: ANSIColor) -> String {
    "\(color.rawValue)\(text)\(ANSIColor.reset.rawValue)"
}

print(colored("Error: file not found", .red))
print(colored("Success!", .green))
```

### Exit Codes

```swift
import ArgumentParser

// ArgumentParser provides ExitCode
throw ExitCode.success          // 0
throw ExitCode.failure          // 1
throw ExitCode.validationFailure // 64
throw ExitCode(42)              // custom
```

### Signal Handling

```swift
import Foundation

let sigintSource = DispatchSource.makeSignalSource(signal: SIGINT, queue: .main)
signal(SIGINT, SIG_IGN) // Ignore default handler
sigintSource.setEventHandler {
    print("\nInterrupted. Cleaning up...")
    // cleanup logic
    Foundation.exit(0)
}
sigintSource.resume()
```

---

## SwifTeaUI — Terminal User Interfaces

[SwifTeaUI](https://github.com/jerihass/SwifTeaUI) is a declarative TUI framework inspired by SwiftUI and Bubble Tea. It uses The Elm Architecture (TEA): Model holds state, Actions describe changes, `update` applies them, `view` renders.

### Platforms

macOS and Linux. Requires Swift 6.0+.

### Core Architecture

```
TUIApp (@main entry point)
  └── TUIScene (state + update + view + key mapping)
        ├── Model (state struct)
        ├── Action (enum of state mutations)
        ├── update(action:) — reducer
        ├── view(model:) — render
        └── mapKeyToAction(_:) — keyboard routing
```

### Minimal Example

```swift
import SwifTeaUI

@main
struct CounterApp: TUIApp {
    var body: some TUIScene { CounterScene() }
}

struct CounterScene: TUIScene {
    typealias Model = CounterState
    typealias Action = CounterState.Action

    var model = CounterState()

    mutating func update(action: Action) {
        switch action {
        case .increment: model.count += 1
        case .decrement: model.count -= 1
        case .quit: break
        }
    }

    func view(model: Model) -> some TUIView {
        VStack(spacing: 1) {
            Text("Counter").bold().foregroundColor(.yellow)
            Text("Count: \(model.count)").foregroundColor(.green)
            Text("(+/- to change, q to quit)").foregroundColor(.cyan)
        }
        .padding(1)
    }

    func mapKeyToAction(_ key: KeyEvent) -> Action? {
        switch key {
        case .char("+"): return .increment
        case .char("-"): return .decrement
        case .char("q"), .escape: return .quit
        default: return nil
        }
    }

    func shouldExit(for action: Action) -> Bool { action == .quit }
}

struct CounterState {
    enum Action { case increment, decrement, quit }
    var count = 0
}
```

### View Components

| Component | Purpose |
|-----------|---------|
| `Text` | Styled text display |
| `VStack` | Vertical layout with spacing |
| `HStack` | Horizontal layout |
| `ZStack` | Z-order layering for overlays |
| `TextField` | Single-line text input |
| `TextEditor` / `TextArea` | Multi-line text editor |
| `Table` | Columnar data with selection |
| `List` | Simple vertical enumeration |
| `ForEach` | Data-driven repetition |
| `Border` | Box drawing around content |
| `ScrollView` | Scrollable viewport |
| `Spinner` | Animated activity indicator (.ascii, .braille, .dots, .line) |
| `ProgressMeter` | Progress bar (`[####----] 50%`) |
| `AdaptiveStack` | Responsive layout by terminal width |
| `MinimumTerminalSize` | Fallback when terminal too small |
| `StatusBar` | Status strip |

### State Management

```swift
// Local value state
@State private var count = 0

// Shared reference model (persists across re-renders)
@StateObject private var model = SharedModel()
@ObservedObject var model: SharedModel

// Focus tracking
@FocusState private var focused: Field?
```

### Focus Navigation

```swift
enum Field: Hashable { case name, email, message }

@FocusState private var focused: Field?
private let ring = FocusRing<Field>([.name, .email, .message])
private let formScope = FocusScope<Field>([.name, .email, .message], wraps: true)

// In view
TextField("Name", text: $name)
    .focused($focused, equals: .name)
TextField("Email", text: $email)
    .focused($focused, equals: .email)

// In mapKeyToAction
case .tab:
    $focused.moveForward(in: formScope)
    return nil
case .backTab:
    $focused.moveBackward(in: formScope)
    return nil
```

### Table with Selection

```swift
Table(
    users,
    selection: .single($selectedUser),
    rowStyle: .stripedRows()
) {
    TableColumn("Name", value: \.name, width: .flex(min: 10, max: 30))
    TableColumn("Email", value: \.email, width: .flex(min: 15, max: 40))
    TableColumn("Role", value: \.role, width: .fixed(12))
}
```

### Async Effects

```swift
// Dispatch an action
SwifTea.dispatch(Action.startLoading)

// Run async work
SwifTea.dispatch(
    Effect<Action>.run { send in
        let data = try await fetchData()
        send(.dataLoaded(data))
    },
    id: "fetch",
    cancelExisting: true
)

// Recurring timer
SwifTea.dispatch(
    Effect<Action>.timer(every: 0.5) { .tick },
    id: "timer"
)

// Cancel
SwifTea.cancelEffects(withID: "fetch")
```

### Styling

```swift
Text("Error!")
    .foregroundColor(.red)
    .bold()

Text("Note")
    .foregroundColor(.cyan)
    .italic()
    .underline()

TextField("Search", text: $query)
    .focused($focused, equals: .search)
    .focusRingStyle(FocusRingBorder(color: .blue))
    .blinkingCursor()
```

### Terminal Awareness

```swift
// Check terminal size
let metrics = TerminalMetrics.current()
// metrics.size.columns, metrics.size.rows, metrics.sizeClass

// Responsive layout
AdaptiveStack(breakpoint: 80,
    expanded: { HStack { sidebar; content } },
    collapsed: { VStack { content } }
)

// Minimum size guard
MinimumTerminalSize(columns: 60, rows: 20) {
    Text("Terminal too small. Please resize.")
} content: {
    MainView()
}
```

### Scene Lifecycle

```swift
struct MyScene: TUIScene {
    // Called once at startup — seed timers, background work
    func initializeEffects() {
        SwifTea.dispatch(Effect<Action>.timer(every: 1.0) { .tick }, id: "clock")
    }

    // Called every frame — use for animation
    func handleFrame(deltaTime: Double) {
        // animation updates
    }

    // React to terminal resize
    func handleTerminalResize(from oldSize: Size, to newSize: Size) {
        SwifTea.dispatch(.resize(newSize))
    }
}
```

---

## Building & Distribution

### Build Release Binary

```bash
# Native release build
swift build -c release

# Universal binary (macOS, both Intel and Apple Silicon)
swift build -c release --arch arm64 --arch x86_64

# Binary location
.build/release/my-tool
```

### Install Locally

```bash
# Copy to a directory in PATH
cp .build/release/my-tool /usr/local/bin/
```

### Homebrew Formula

```ruby
class MyTool < Formula
  desc "Description of my tool"
  homepage "https://github.com/user/my-tool"
  url "https://github.com/user/my-tool/archive/refs/tags/v1.0.0.tar.gz"
  sha256 "abc123..."
  license "MIT"

  depends_on xcode: ["15.0", :build]

  def install
    system "swift", "build", "-c", "release", "--disable-sandbox"
    bin.install ".build/release/my-tool"
  end

  test do
    assert_match "my-tool", shell_output("#{bin}/my-tool --help")
  end
end
```

### Mint (Swift-specific package manager)

Ensure the `Package.swift` has an executable product and users can install via:

```bash
mint install user/my-tool
```

---

## Patterns

### ✅ Use stderr for progress, stdout for data

```swift
printError("Processing...")  // stderr
print(result)                // stdout — safe to pipe
```

### ❌ Don't mix progress output into stdout

```swift
// Breaks piping: `my-tool | jq` will fail on "Processing..."
print("Processing...")
print(result)
```

### ✅ Use AsyncParsableCommand for any I/O-bound work

```swift
struct Fetch: AsyncParsableCommand {
    func run() async throws {
        let data = try await URLSession.shared.data(from: url)
    }
}
```

### ❌ Don't block the main thread with synchronous network calls

```swift
struct Fetch: ParsableCommand {
    func run() throws {
        // Blocks, no timeout control, poor cancellation
        let data = try Data(contentsOf: url)
    }
}
```

### ✅ Provide structured output formats for scriptability

```swift
@Option var format: OutputFormat = .text

func run() throws {
    switch globals.format {
    case .json: print(try JSONEncoder().encode(result))
    case .text: print(result.description)
    }
}
```

### ❌ Don't output only human-readable text with no machine option

```swift
// Other tools can't parse this
print("Found 42 results in 3 files")
```

### ✅ Handle SIGINT for cleanup in long-running tools

```swift
sigintSource.setEventHandler {
    cleanup()
    Foundation.exit(0)
}
```

### ❌ Don't let Ctrl+C leave temporary files behind

```swift
// No signal handling — temp files leak on interrupt
func run() throws {
    let tmp = createTempFile()
    processData()  // Ctrl+C here leaves tmp orphaned
    deleteTempFile(tmp)
}
```
