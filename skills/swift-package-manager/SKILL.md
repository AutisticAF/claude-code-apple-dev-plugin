---
name: swift-package-manager
description: Swift Package Manager patterns for creating packages, multi-module projects, dependency management, plugins, and local packages. Use when structuring Swift projects with SPM.
---

# Swift Package Manager

Comprehensive guidance for structuring Swift projects with SPM, from single libraries to multi-module apps.

## When This Skill Activates

Use this skill when the user:
- Creates a new Swift package (library or executable)
- Structures a multi-module project with SPM
- Adds or manages remote/local dependencies
- Configures build tool plugins or command plugins
- Works with binary targets or XCFrameworks
- Needs help with Package.swift syntax or version resolution
- Wants to modularize an Xcode project using local packages
- Asks about package resources (Bundle.module)
- Compares SPM vs CocoaPods vs Carthage

## Decision Tree

### Which Dependency Manager?

| Factor | SPM | CocoaPods | Carthage |
|--------|-----|-----------|----------|
| Apple-native integration | Yes | No | No |
| No extra tooling | Yes | Requires Ruby | Requires Homebrew |
| Binary distribution | XCFramework | .framework | .framework |
| Xcode integration | Built-in | .xcworkspace | Manual |
| Active development | Yes | Maintenance mode | Low activity |

**Recommendation**: Use SPM unless a dependency only ships a Podspec.

### Single Package vs Multi-Module?

- **Single package**: Standalone library or tool with one public API surface
- **Multi-module**: App with 3+ feature areas, shared utilities, or a need for faster incremental builds and clearer dependency boundaries

## API Availability by Swift Tools Version

| Swift Tools Version | Key Features |
|---------------------|-------------|
| 5.4 | Executable targets, `@main` entry point |
| 5.5 | `#if canImport`, improved dependency resolution |
| 5.6 | Build tool plugins, command plugins |
| 5.7 | Package registry support, `swift package init --type macro` prep |
| 5.8 | Improved `Target.dependency` conditions |
| 5.9 | Macros as package targets, `CompilerPluginSupport` |
| 5.10 | Strict concurrency defaults in package manifests |
| 6.0 | Swift Testing integration, trait-based test configuration |
| 6.1 | Package collections improvements, registry enhancements |

## Package.swift Anatomy

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "MyLibrary",
    platforms: [
        .iOS(.v17),
        .macOS(.v14),
        .watchOS(.v10),
        .tvOS(.v17),
        .visionOS(.v1)
    ],
    products: [
        .library(name: "MyLibrary", targets: ["MyLibrary"]),
        .executable(name: "my-tool", targets: ["MyTool"]),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser", from: "1.3.0"),
    ],
    targets: [
        .target(
            name: "MyLibrary",
            dependencies: [],
            resources: [.process("Resources")]
        ),
        .executableTarget(
            name: "MyTool",
            dependencies: [
                .product(name: "ArgumentParser", library: "swift-argument-parser"),
            ]
        ),
        .testTarget(
            name: "MyLibraryTests",
            dependencies: ["MyLibrary"]
        ),
    ]
)
```

### Key Components

- **products**: What consumers see --- `.library` (static/dynamic) or `.executable`
- **targets**: Build units with source in `Sources/<TargetName>/`
- **dependencies**: External packages resolved by SPM
- **platforms**: Minimum deployment targets (omit for tools-only packages)

## Creating a Library Package

```bash
mkdir MyLibrary && cd MyLibrary
swift package init --type library --name MyLibrary
```

Directory layout:

```
MyLibrary/
├── Package.swift
├── Sources/
│   └── MyLibrary/
│       └── MyLibrary.swift
└── Tests/
    └── MyLibraryTests/
        └── MyLibraryTests.swift
```

## Creating an Executable Package

```bash
swift package init --type executable --name my-tool
```

```swift
// Sources/my-tool/main.swift  (or use @main with ArgumentParser)
import ArgumentParser

@main
struct MyTool: ParsableCommand {
    @Argument var name: String

    func run() throws {
        print("Hello, \(name)!")
    }
}
```

## Multi-Module Project Structure

Split features into targets with explicit dependency edges:

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [.iOS(.v17)],
    products: [
        .library(name: "AppFeatures", targets: ["HomeFeature", "ProfileFeature"]),
    ],
    dependencies: [],
    targets: [
        // Shared utilities --- no upstream dependencies
        .target(name: "SharedModels"),
        .target(name: "Networking", dependencies: ["SharedModels"]),

        // Feature modules depend on shared layers
        .target(name: "HomeFeature", dependencies: ["Networking", "SharedModels"]),
        .target(name: "ProfileFeature", dependencies: ["Networking", "SharedModels"]),

        // Tests per module
        .testTarget(name: "HomeFeatureTests", dependencies: ["HomeFeature"]),
        .testTarget(name: "ProfileFeatureTests", dependencies: ["ProfileFeature"]),
    ]
)
```

```
Sources/
├── SharedModels/
├── Networking/
├── HomeFeature/
└── ProfileFeature/
```

## Local Packages for Modularization

Use path-based dependencies to split an Xcode project without publishing:

```swift
// In the app's Package.swift or Xcode project SPM dependencies
.package(path: "../CoreKit"),
.package(path: "../DesignSystem"),
```

In Xcode: File > Add Package Dependencies > Add Local > select folder.

```
MyApp/
├── MyApp.xcodeproj
├── Packages/
│   ├── CoreKit/
│   │   ├── Package.swift
│   │   └── Sources/
│   └── DesignSystem/
│       ├── Package.swift
│       └── Sources/
```

## Remote Dependencies and Version Rules

```swift
dependencies: [
    // Semantic versioning (recommended)
    .package(url: "https://github.com/apple/swift-log", from: "1.5.0"),

    // Up to next major (same as from:)
    .package(url: "https://github.com/apple/swift-nio", .upToNextMajor(from: "2.60.0")),

    // Up to next minor
    .package(url: "https://github.com/pointfreeco/swift-composable-architecture",
             .upToNextMinor(from: "1.9.0")),

    // Exact version (avoid in libraries --- locks consumers)
    .package(url: "https://github.com/example/pinned-lib", exact: "3.2.1"),

    // Branch (CI/development only)
    .package(url: "https://github.com/example/bleeding-edge", branch: "main"),

    // Specific commit
    .package(url: "https://github.com/example/audited-lib",
             revision: "a1b2c3d4e5f6"),
]
```

## Resources: Bundle.module

```swift
.target(
    name: "MyFeature",
    dependencies: [],
    resources: [
        .process("Resources"),     // Optimized by platform (images, strings)
        .copy("Fixtures")          // Copied as-is (JSON test data, ML models)
    ]
)
```

Access at runtime:

```swift
let image = UIImage(named: "icon", in: .module, compatibleWith: nil)
let url = Bundle.module.url(forResource: "config", withExtension: "json")!
let data = try Data(contentsOf: url)
```

**`.process` vs `.copy`**: Use `.process` for assets the build system should optimize (images, strings, asset catalogs). Use `.copy` for files that must remain unchanged (JSON fixtures, ML models, scripts).

## Plugins

### Build Tool Plugin

Runs automatically during the build (e.g., code generation):

```swift
.plugin(
    name: "GenerateResources",
    capability: .buildTool(),
    dependencies: ["resource-generator"]
),
.executableTarget(name: "resource-generator"),
```

Plugin implementation at `Plugins/GenerateResources/plugin.swift`:

```swift
import PackagePlugin

@main
struct GenerateResources: BuildToolPlugin {
    func createBuildCommands(context: PluginContext, target: Target) throws -> [Command] {
        let tool = try context.tool(named: "resource-generator")
        let output = context.pluginWorkDirectoryURL.appending(path: "Generated.swift")
        return [
            .buildCommand(
                displayName: "Generate resources",
                executable: tool.url,
                arguments: [target.directoryURL.path(), output.path()],
                outputFiles: [output]
            )
        ]
    }
}
```

### Command Plugin

Runs on demand via `swift package <command>`:

```swift
.plugin(
    name: "FormatCode",
    capability: .command(
        intent: .sourceCodeFormatting(),
        permissions: [.writeToPackageDirectory(reason: "Format source files")]
    )
)
```

## Conditional Compilation in Package.swift

```swift
var targets: [Target] = [
    .target(name: "Core"),
]

#if os(macOS)
targets.append(.executableTarget(name: "CLI", dependencies: ["Core"]))
#endif

#if compiler(>=6.0)
targets.append(.target(name: "MacroSupport", dependencies: [
    .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
]))
#endif

let package = Package(
    name: "CrossPlatform",
    targets: targets
)
```

## Testing Packages

```swift
.testTarget(
    name: "MyLibraryTests",
    dependencies: [
        "MyLibrary",
        .product(name: "CustomDump", package: "swift-custom-dump"),
    ],
    resources: [.copy("Fixtures")]
)
```

```bash
swift test                           # Run all tests
swift test --filter MyLibraryTests   # Run one test target
swift test --parallel                # Parallel execution
```

With Swift Testing (6.0+):

```swift
import Testing
@testable import MyLibrary

@Test("Parses valid input")
func parseValidInput() {
    let result = Parser.parse("hello")
    #expect(result == .success("hello"))
}
```

## Binary Targets and XCFramework Distribution

```swift
.binaryTarget(
    name: "Analytics",
    url: "https://example.com/Analytics-1.2.0.xcframework.zip",
    checksum: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
),
.binaryTarget(
    name: "LocalSDK",
    path: "Frameworks/LocalSDK.xcframework"
),
```

Create an XCFramework:

```bash
xcodebuild archive -scheme MyLib -destination "generic/platform=iOS" \
    -archivePath build/ios SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES
xcodebuild archive -scheme MyLib -destination "generic/platform=iOS Simulator" \
    -archivePath build/sim SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES
xcodebuild -create-xcframework \
    -archive build/ios.xcarchive -framework MyLib.framework \
    -archive build/sim.xcarchive -framework MyLib.framework \
    -output MyLib.xcframework
```

## Package Collections and Registry

```bash
# Add a collection (curated list of packages)
swift package-collection add https://swiftpackageindex.com/collection.json

# Search collections
swift package-collection search --keywords networking

# Use a registry (Swift 5.7+)
swift package-registry set https://registry.example.com
```

## Patterns

### ✅ Good Patterns

```swift
// ✅ Explicit platform requirements
platforms: [.iOS(.v17), .macOS(.v14)]

// ✅ Semantic version ranges for dependencies
.package(url: "https://github.com/apple/swift-log", from: "1.5.0")

// ✅ Separate targets per feature for clear boundaries
.target(name: "Networking", dependencies: ["SharedModels"])
.target(name: "HomeFeature", dependencies: ["Networking"])

// ✅ Use .product(name:package:) when target name differs from package name
.product(name: "ArgumentParser", package: "swift-argument-parser")

// ✅ Process resources for platform optimization
resources: [.process("Resources")]
```

### ❌ Bad Patterns

```swift
// ❌ Pinning exact versions in libraries (blocks consumer resolution)
.package(url: "https://github.com/example/lib", exact: "2.0.0")

// ❌ Using branch dependencies in released packages
.package(url: "https://github.com/example/lib", branch: "main")

// ❌ One massive target with everything
.target(name: "MyApp", dependencies: [], path: "Sources")

// ❌ Forgetting to add test targets
// (every target should have a corresponding testTarget)

// ❌ Importing a whole package when you only need one product
// Use .product(name:package:) to pick the specific product
dependencies: ["swift-composable-architecture"]  // ambiguous if multiple products
```
