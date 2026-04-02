---
name: swift-macros
description: Swift macro system including freestanding and attached macros, macro roles, testing macros, and common patterns. Use when creating, debugging, or understanding Swift macros.
---

> **First step:** Tell the user: "swift-macros skill loaded."

# Swift Macros

Swift macros generate code at compile time by operating on the abstract syntax tree (AST) via SwiftSyntax. They eliminate boilerplate while keeping generated code visible and debuggable. Macros run as compiler plugins in a sandbox -- they can only read the syntax they receive and produce new syntax in return.

## When This Skill Activates

- User wants to create a custom Swift macro
- User asks about freestanding or attached macros
- User needs help choosing the right macro role
- User is setting up a Swift macro package with SwiftSyntax
- User wants to test macro expansions
- User is debugging macro compilation or expansion errors
- User asks about `#externalMacro`, `@freestanding`, or `@attached`
- User wants to reduce boilerplate with compile-time code generation

## Decision Tree: Choosing a Macro Role

```
Do you need to create a new declaration or transform an expression?
├─ YES: Create a new expression from inputs?
│  └─ @freestanding(expression) -- produces a value
├─ YES: Create one or more new declarations at top level or in a type?
│  └─ @freestanding(declaration) -- produces declarations
├─ NO: You want to augment an existing declaration
│  ├─ Add new declarations alongside it?
│  │  └─ @attached(peer) -- adds sibling declarations
│  ├─ Add members inside a type?
│  │  └─ @attached(member) -- adds properties, methods, inits
│  ├─ Add attributes to all members of a type?
│  │  └─ @attached(memberAttribute) -- annotates each member
│  ├─ Add computed property accessors?
│  │  └─ @attached(accessor) -- adds get/set/willSet/didSet
│  ├─ Add protocol conformances and extensions?
│  │  └─ @attached(extension) -- adds extension blocks
│  ├─ Provide or replace a function/initializer body?
│  │  └─ @attached(body) -- generates function body
│  └─ Inject code at the start of a function body?
│     └─ @attached(preamble) -- inserts code before existing body
```

## API Availability

| Feature | Minimum Version | Notes |
|---------|----------------|-------|
| Freestanding macros | Swift 5.9 / Xcode 15 | `@freestanding(expression)`, `@freestanding(declaration)` |
| Attached macros | Swift 5.9 / Xcode 15 | `@attached(peer, member, accessor, memberAttribute, extension)` |
| `@attached(body)` | Swift 6.0 / Xcode 16 | Generates function/initializer bodies |
| `@attached(preamble)` | Swift 6.0 / Xcode 16 | Injects code at start of function body |
| SwiftSyntax 509 | Swift 5.9 | Matches Swift language version |
| SwiftSyntax 600 | Swift 6.0 | Updated AST nodes |

---

## Freestanding Macros

### @freestanding(expression) -- Produces a Value

```swift
// Declaration (in your library target)
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) =
    #externalMacro(module: "MyMacroMacros", type: "StringifyMacro")

// Usage
let (result, code) = #stringify(2 + 3)
// result == 5, code == "2 + 3"
```

### @freestanding(declaration) -- Produces Declarations

```swift
// Declaration
@freestanding(declaration, names: arbitrary)
public macro makeEnum(name: String, cases: [String]) =
    #externalMacro(module: "MyMacroMacros", type: "MakeEnumMacro")

// Usage
#makeEnum(name: "Direction", cases: ["north", "south", "east", "west"])
// Expands to:
// enum Direction { case north, south, east, west }
```

---

## Attached Macros

### @attached(peer) -- Adds Sibling Declarations

```swift
@attached(peer, names: prefixed(_))
public macro AddCompletionHandler() =
    #externalMacro(module: "MyMacroMacros", type: "AddCompletionHandlerMacro")

// Usage
@AddCompletionHandler
func fetchUser(id: Int) async throws -> User { ... }
// Expands a sibling: func _fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void)
```

### @attached(member) -- Adds Members Inside a Type

```swift
@attached(member, names: named(init))
public macro AddInit() =
    #externalMacro(module: "MyMacroMacros", type: "AddInitMacro")

// Usage
@AddInit
struct User {
    let name: String
    let age: Int
}
// Expands: init(name: String, age: Int) { self.name = name; self.age = age }
```

### @attached(accessor) -- Adds Property Accessors

```swift
@attached(accessor, names: named(get), named(set))
public macro DictionaryBacked(key: String) =
    #externalMacro(module: "MyMacroMacros", type: "DictionaryBackedMacro")

// Usage
struct Settings {
    var storage: [String: Any] = [:]

    @DictionaryBacked(key: "user_name")
    var userName: String
    // Expands get/set that read/write storage["user_name"]
}
```

### @attached(memberAttribute) -- Annotates Each Member

```swift
@attached(memberAttribute)
public macro CodableKeys() =
    #externalMacro(module: "MyMacroMacros", type: "CodableKeysMacro")

// Usage -- adds @CodingKey to all stored properties
@CodableKeys
struct APIResponse: Codable {
    var userId: Int     // gets @CodingKey("user_id")
    var userName: String // gets @CodingKey("user_name")
}
```

### @attached(extension) -- Adds Protocol Conformances

```swift
@attached(extension, conformances: Equatable, Hashable, names: named(==), named(hash))
public macro AutoEquatable() =
    #externalMacro(module: "MyMacroMacros", type: "AutoEquatableMacro")

// Usage
@AutoEquatable
struct Point {
    let x: Double
    let y: Double
}
// Expands an extension with Equatable and Hashable conformance
```

### @attached(body) -- Generates a Function Body (Swift 6.0+)

```swift
@attached(body)
public macro RemoteCall(endpoint: String) =
    #externalMacro(module: "MyMacroMacros", type: "RemoteCallMacro")

// Usage
@RemoteCall(endpoint: "/users")
func fetchUsers() async throws -> [User]
// Macro generates the entire function body
```

### @attached(preamble) -- Injects Code Before Body (Swift 6.0+)

```swift
@attached(preamble)
public macro Traced() =
    #externalMacro(module: "MyMacroMacros", type: "TracedMacro")

// Usage
@Traced
func processOrder(_ order: Order) {
    // Macro injects: os_log("entering processOrder(_:)")
    validate(order)
    submit(order)
}
```

---

## Macro Package Setup

Create a macro package with `File > New > Package` in Xcode or via the command line:

```bash
mkdir MyMacro && cd MyMacro
swift package init --type macro
```

### Package.swift

```swift
// swift-tools-version: 5.9
import PackageDescription
import CompilerPluginSupport

let package = Package(
    name: "MyMacro",
    platforms: [.macOS(.v10_15), .iOS(.v13)],
    products: [
        .library(name: "MyMacro", targets: ["MyMacro"]),
    ],
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-syntax.git", from: "509.0.0"),
    ],
    targets: [
        // The macro implementation (compiler plugin, runs at build time)
        .macro(
            name: "MyMacroMacros",
            dependencies: [
                .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
            ]
        ),
        // The library that exposes macro declarations to clients
        .target(name: "MyMacro", dependencies: ["MyMacroMacros"]),
        // Tests
        .testTarget(
            name: "MyMacroTests",
            dependencies: [
                "MyMacroMacros",
                .product(name: "SwiftSyntaxMacrosTestSupport", package: "swift-syntax"),
            ]
        ),
    ]
)
```

### Three-Target Pattern

| Target | Purpose | Contains |
|--------|---------|----------|
| `MyMacroMacros` | `.macro` target -- compiler plugin | `ExpressionMacro`, `MemberMacro` conformances |
| `MyMacro` | `.target` -- public API | `@freestanding`/`@attached` declarations with `#externalMacro` |
| `MyMacroTests` | `.testTarget` | `assertMacroExpansion` tests |

---

## Macro Implementation with SwiftSyntax

### Expression Macro Example: #URL

```swift
import SwiftSyntax
import SwiftSyntaxMacros
import SwiftCompilerPlugin

public struct URLMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) throws -> ExprSyntax {
        guard let argument = node.arguments.first?.expression,
              let segments = argument.as(StringLiteralExprSyntax.self)?.segments,
              segments.count == 1,
              case .stringSegment(let literalSegment) = segments.first
        else {
            throw MacroError.message("#URL requires a static string literal")
        }

        guard let _ = URL(string: literalSegment.content.text) else {
            throw MacroError.message("Invalid URL: \(literalSegment.content.text)")
        }

        return "URL(string: \(argument))!"
    }
}

// Register the plugin
@main
struct MyMacroPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
        URLMacro.self,
    ]
}
```

### Member Macro Example: @AddInit

```swift
public struct AddInitMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        let members = declaration.memberBlock.members
        let storedProperties = members.compactMap { member -> (name: String, type: String)? in
            guard let property = member.decl.as(VariableDeclSyntax.self),
                  let binding = property.bindings.first,
                  let name = binding.pattern.as(IdentifierPatternSyntax.self)?.identifier.text,
                  let type = binding.typeAnnotation?.type.trimmedDescription,
                  binding.accessorBlock == nil
            else { return nil }
            return (name, type)
        }

        let params = storedProperties
            .map { "\($0.name): \($0.type)" }
            .joined(separator: ", ")
        let assignments = storedProperties
            .map { "self.\($0.name) = \($0.name)" }
            .joined(separator: "\n        ")

        return [
            """
            init(\(raw: params)) {
                \(raw: assignments)
            }
            """
        ]
    }
}
```

### Peer Macro Example: @EnumSubset

```swift
public struct EnumSubsetMacro: PeerMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingPeersOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        // Implementation that generates a subset enum alongside the original
        // ...
        return []
    }
}
```

### Error Reporting

```swift
enum MacroError: CustomStringConvertible, Error {
    case message(String)
    var description: String {
        switch self {
        case .message(let text): return text
        }
    }
}

// For richer diagnostics with fix-its:
import SwiftDiagnostics

let diagnostic = Diagnostic(
    node: node,
    message: MyDiagnosticMessage.mustBeStruct,
    fixIt: FixIt(message: MyFixItMessage.addStructKeyword, changes: [
        .replace(oldNode: Syntax(node), newNode: Syntax(corrected))
    ])
)
context.diagnose(diagnostic)
```

---

## Testing Macros

### assertMacroExpansion

```swift
import SwiftSyntaxMacrosTestSupport
import XCTest

final class URLMacroTests: XCTestCase {
    let testMacros: [String: Macro.Type] = [
        "URL": URLMacro.self,
    ]

    func testValidURL() throws {
        assertMacroExpansion(
            """
            let url = #URL("https://apple.com")
            """,
            expandedSource: """
            let url = URL(string: "https://apple.com")!
            """,
            macros: testMacros
        )
    }

    func testInvalidURL() throws {
        assertMacroExpansion(
            """
            let url = #URL("not a url ://")
            """,
            expandedSource: """
            let url = #URL("not a url ://")
            """,
            diagnostics: [
                DiagnosticSpec(message: "Invalid URL: not a url ://", line: 1, column: 11)
            ],
            macros: testMacros
        )
    }
}
```

### Testing Attached Macros

```swift
func testAddInit() throws {
    let testMacros: [String: Macro.Type] = [
        "AddInit": AddInitMacro.self,
    ]

    assertMacroExpansion(
        """
        @AddInit
        struct User {
            let name: String
            let age: Int
        }
        """,
        expandedSource: """
        struct User {
            let name: String
            let age: Int

            init(name: String, age: Int) {
                self.name = name
                self.age = age
            }
        }
        """,
        macros: testMacros
    )
}
```

---

## Debugging Macros

### Expand in Tests (Preferred)

Write `assertMacroExpansion` tests first. They run the macro in-process and show exact expansion output with diagnostics. This is the fastest feedback loop.

### Xcode: Right-Click > Expand Macro

In Xcode, right-click any macro call site and select "Expand Macro" to see the generated code inline.

### Compiler Flag

```bash
# Dump all macro expansions during build
swift build -Xswiftc -dump-macro-expansions
```

### Print Debugging in Tests

```swift
// Inside your macro implementation, during test runs:
print(declaration.debugDescription)  // Full AST dump
print(declaration.trimmedDescription) // Cleaned source text
```

---

## Common Patterns

### Combining Multiple Roles

A single macro can adopt multiple roles:

```swift
@attached(member, names: named(init), named(CodingKeys))
@attached(extension, conformances: Codable)
public macro AutoCodable() =
    #externalMacro(module: "MyMacroMacros", type: "AutoCodableMacro")
```

The implementation conforms to both `MemberMacro` and `ExtensionMacro`.

### Names Parameter

The `names:` parameter tells the compiler what declarations the macro will produce, so it can resolve references:

- `names: named(init)` -- will produce an `init`
- `names: prefixed(_)` -- will produce names starting with `_`
- `names: suffixed(Defaults)` -- will produce names ending with `Defaults`
- `names: arbitrary` -- may produce any name (use sparingly; hurts incremental compilation)

---

## Patterns: Good and Bad

### ✅ Good Patterns

```swift
// ✅ Validate inputs at compile time
guard let literal = argument.as(StringLiteralExprSyntax.self) else {
    throw MacroError.message("Argument must be a string literal")
}

// ✅ Use assertMacroExpansion for every macro
func testMyMacro() { assertMacroExpansion(...) }

// ✅ Keep macro logic small; delegate to helper functions
// ✅ Report clear diagnostics with context.diagnose()
// ✅ Specify exact names: parameter instead of arbitrary
// ✅ Use SwiftSyntaxBuilder for constructing complex syntax trees
// ✅ Test error cases and diagnostic messages, not just happy paths
```

### ❌ Bad Patterns

```swift
// ❌ String interpolation for building code -- fragile, no validation
let code = "func \(name)() { return \(value) }"

// ❌ Using arbitrary names when specific names are known
@attached(member, names: arbitrary)  // Bad if you only add init
@attached(member, names: named(init)) // Good

// ❌ Skipping error messages -- the user gets a cryptic compiler error
throw SomeError() // No context

// ❌ Side effects in macros -- they run in a sandbox
try FileManager.default.createFile(...)  // Will fail

// ❌ Relying on runtime values -- macros only see syntax
guard runtimeConfig.isEnabled else { ... } // Cannot access runtime state

// ❌ Generating massive amounts of code -- keep expansions focused
// ❌ Forgetting to register macros in CompilerPlugin.providingMacros
```
