# TOON Format & toon-swift

## What Is TOON?

TOON (Token-Oriented Object Notation) is a compact, human-readable serialization format designed to minimize token usage in LLM prompts. It encodes the JSON data model losslessly using indentation-based syntax and tabular arrays that eliminate repeated key names.

**Spec version:** 3.0 (2025-11-24)
**File extension:** `.toon`
**Media type:** `text/toon`
**Encoding:** UTF-8 with LF line endings

### JSON vs TOON Comparison

JSON:
```json
{
  "users": [
    {"id": 1, "name": "Ada", "role": "admin", "active": true},
    {"id": 2, "name": "Bob", "role": "editor", "active": false},
    {"id": 3, "name": "Cat", "role": "viewer", "active": true}
  ],
  "count": 3
}
```

TOON (30-60% fewer tokens):
```toon
users[3]{id,name,role,active}:
  1,Ada,admin,true
  2,Bob,editor,false
  3,Cat,viewer,true

count: 3
```

The token savings come from: no braces/brackets, no quoted keys, no repeated field names in arrays of objects, and minimal quoting of string values.

## TOON Syntax Overview

### Objects

Key-value pairs with indentation for nesting:

```toon
name: Ada Lovelace
age: 36
address:
  street: 123 Main St
  city: London
```

### Primitive Arrays

Inline with declared length:

```toon
tags[3]: swift,codable,toon
empty[0]:
```

### Tabular Arrays (Signature Feature)

Arrays of uniform objects encoded as a header + rows:

```toon
items[2]{sku,name,qty,price}:
  A1,Widget,2,9.99
  B2,Gadget,1,14.5
```

The `[2]` declares the row count (must match). The `{sku,name,qty,price}` declares the fields. Each row provides values in field order.

### String Quoting

Strings are unquoted by default. Quoting is required when a value:
- Is empty, or equals `true`, `false`, `null`
- Looks numeric (e.g., `"042"`)
- Contains structural characters (`:`, `"`, `\`, `[`, `]`, `{`, `}`)
- Has leading/trailing whitespace
- Starts with `-`

Only five escapes: `\\`, `\"`, `\n`, `\r`, `\t`.

### Delimiters

Three delimiter options for arrays and tabular rows:

| Delimiter | Symbol | Header syntax | Use when |
|-----------|--------|---------------|----------|
| Comma | `,` | `[N]` or `{f1,f2}` | Default — values don't contain commas |
| Tab | `\t` | `[N\t]` or `{f1\tf2}` | Values contain commas |
| Pipe | `\|` | `[N\|]` or `{f1\|f2}` | Values contain commas and tabs |

### Key Folding

Single-child nested objects can collapse into dotted paths:

```toon
database.connection.host: localhost
database.connection.port: 5432
database.name: myapp
```

Expands to:
```json
{"database": {"connection": {"host": "localhost", "port": 5432}, "name": "myapp"}}
```

## toon-swift Library

### Installation

**Swift Package Manager** — add to `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/toon-format/toon-swift", from: "0.3.0")
]
```

Target dependency:

```swift
.target(name: "MyApp", dependencies: [
    .product(name: "ToonFormat", package: "toon-swift")
])
```

**Platform requirements:** iOS 13+, macOS 10.15+, watchOS 6+, tvOS 13+, visionOS 1+. Swift 6.0+ / Xcode 16+.

### Basic Usage

`TOONEncoder` and `TOONDecoder` are drop-in replacements for `JSONEncoder`/`JSONDecoder` — they work with any `Codable` type:

```swift
import ToonFormat

struct User: Codable {
    let id: Int
    let name: String
    let tags: [String]
    let active: Bool
}

// Encode to TOON
let encoder = TOONEncoder()
let data = try encoder.encode(User(id: 1, name: "Ada", tags: ["admin", "dev"], active: true))
// Output:
// id: 1
// name: Ada
// tags[2]: admin,dev
// active: true

// Decode from TOON
let decoder = TOONDecoder()
let user = try decoder.decode(User.self, from: data)
```

### Tabular Arrays (Automatic)

When encoding an array of structs where all instances share the same keys and all values are primitives, `TOONEncoder` automatically uses tabular format:

```swift
struct Score: Codable {
    let player: String
    let points: Int
    let level: Int
}

let scores = [
    Score(player: "Ada", points: 1200, level: 5),
    Score(player: "Bob", points: 980, level: 4),
    Score(player: "Cat", points: 1450, level: 6),
]

let data = try TOONEncoder().encode(scores)
// Output (root array, tabular):
// [3]{player,points,level}:
//   Ada,1200,5
//   Bob,980,4
//   Cat,1450,6
```

No special configuration needed — tabular encoding is applied automatically when the data is uniform.

### TOONEncoder Configuration

```swift
let encoder = TOONEncoder()

// Indentation (default: 2 spaces)
encoder.indent = 4

// Delimiter (default: .comma)
encoder.delimiter = .tab    // use when values contain commas
encoder.delimiter = .pipe   // use when values contain commas and tabs

// Key folding (default: .disabled)
encoder.keyFolding = .safe  // collapse single-child nesting into dotted paths
encoder.flattenDepth = 3    // max segments to fold (default: .max)

// Non-conforming floats (default: .null)
encoder.nonConformingFloatEncodingStrategy = .null     // Inf/NaN → null
encoder.nonConformingFloatEncodingStrategy = .throw    // error on Inf/NaN
encoder.nonConformingFloatEncodingStrategy = .convertToString(
    positiveInfinity: "Infinity",
    negativeInfinity: "-Infinity",
    nan: "NaN"
)

// Negative zero (default: .normalize)
encoder.negativeZeroEncodingStrategy = .normalize  // -0.0 → 0
encoder.negativeZeroEncodingStrategy = .preserve   // keep -0.0

// Depth limit (default: 32)
encoder.limits = TOONEncoder.EncodingLimits(maxDepth: 64)
encoder.limits = .unlimited
```

### TOONDecoder Configuration

```swift
let decoder = TOONDecoder()

// Path expansion for dotted keys (default: .automatic)
decoder.expandPaths = .automatic  // expand dotted keys, graceful fallback on conflicts
decoder.expandPaths = .disabled   // treat dotted keys as literal strings
decoder.expandPaths = .safe       // strict expansion, throw on collision

// Resource limits (default: 10 MB, depth 32, 10K keys, 100K array items)
decoder.limits = TOONDecoder.DecodingLimits(
    maxInputSize: 1_048_576,
    maxDepth: 64,
    maxObjectKeys: 1_000,
    maxArrayLength: 10_000
)
decoder.limits = .unlimited
```

### Error Handling

`TOONDecoder` throws `TOONDecodingError` with detailed context:

```swift
do {
    let model = try TOONDecoder().decode(MyModel.self, from: data)
} catch let error as TOONDecodingError {
    switch error {
    case .countMismatch(let expected, let actual, let line):
        print("Row count mismatch at line \(line): expected \(expected), got \(actual)")
    case .fieldCountMismatch(let expected, let actual, let line):
        print("Field count mismatch at line \(line): expected \(expected), got \(actual)")
    case .invalidIndentation(let line, let message):
        print("Bad indentation at line \(line): \(message)")
    case .pathCollision(let path, let line):
        print("Key folding collision at '\(path)', line \(line)")
    case .inputTooLarge(let size, let limit):
        print("Input \(size) bytes exceeds limit \(limit)")
    case .typeMismatch(let expected, let actual):
        print("Type mismatch: expected \(expected), got \(actual)")
    case .keyNotFound(let key):
        print("Missing key: \(key)")
    default:
        print("TOON decoding error: \(error)")
    }
}
```

### Special Type Handling

toon-swift handles Foundation types via reflection before standard Codable encoding:

| Type | TOON encoding |
|------|---------------|
| `Date` | ISO 8601 string |
| `URL` | `absoluteString` |
| `Data` | Base64 string |

### Key Ordering

- **Structs:** Key order matches property declaration order (via Codable synthesis)
- **Dictionaries:** Keys are sorted lexicographically for deterministic output

## Patterns

### LLM Prompt Context

Use TOON to pack more structured data into LLM context windows:

```swift
import ToonFormat

func buildPromptContext(users: [User], tasks: [Task]) throws -> String {
    let encoder = TOONEncoder()
    encoder.keyFolding = .safe

    let userData = try encoder.encode(users)
    let taskData = try encoder.encode(tasks)

    return """
    Here is the current state:

    Users:
    \(String(data: userData, encoding: .utf8)!)

    Tasks:
    \(String(data: taskData, encoding: .utf8)!)
    """
}
```

### Dual-Format Serialization

Support both JSON (for APIs) and TOON (for LLM contexts / storage) with the same `Codable` types:

```swift
import Foundation
import ToonFormat

enum SerializationFormat {
    case json
    case toon
}

struct FormatConverter {
    static func encode<T: Encodable>(_ value: T, as format: SerializationFormat) throws -> Data {
        switch format {
        case .json:
            let encoder = JSONEncoder()
            encoder.outputFormatting = [.sortedKeys]
            encoder.dateEncodingStrategy = .iso8601
            return try encoder.encode(value)
        case .toon:
            let encoder = TOONEncoder()
            encoder.keyFolding = .safe
            return try encoder.encode(value)
        }
    }

    static func decode<T: Decodable>(_ type: T.Type, from data: Data, format: SerializationFormat) throws -> T {
        switch format {
        case .json:
            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .iso8601
            return try decoder.decode(type, from: data)
        case .toon:
            return try TOONDecoder().decode(type, from: data)
        }
    }
}
```

### JSON-to-TOON Migration

Since both formats use Swift's Codable, migration is straightforward — decode from one, encode to the other:

```swift
import Foundation
import ToonFormat

func convertJSONToTOON(_ jsonData: Data) throws -> Data {
    // Decode as generic JSON structure
    let value = try JSONDecoder().decode(AnyCodableValue.self, from: jsonData)
    return try TOONEncoder().encode(value)
}

func convertTOONToJSON(_ toonData: Data) throws -> Data {
    let value = try TOONDecoder().decode(AnyCodableValue.self, from: toonData)
    let encoder = JSONEncoder()
    encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
    return try encoder.encode(value)
}
```

For typed models, simply decode with one decoder and encode with the other:

```swift
let users = try JSONDecoder().decode([User].self, from: jsonData)
let toonData = try TOONEncoder().encode(users)
```

## TOON Format Quick Reference

| Feature | Syntax | Example |
|---------|--------|---------|
| Object | `key: value` (indented nesting) | `name: Ada` |
| Primitive array | `key[N]: v1,v2,v3` | `tags[2]: swift,ios` |
| Empty array | `key[0]:` | `items[0]:` |
| Tabular array | `key[N]{f1,f2}: rows...` | `users[2]{id,name}:` |
| Tab delimiter | `[N\t]` / `{f1\tf2}` | `items[2\t]{a\tb}:` |
| Pipe delimiter | `[N\|]` / `{f1\|f2}` | `items[2\|]{a\|b}:` |
| Dotted key (folded) | `a.b.c: value` | `db.host: localhost` |
| Quoted string | `"value with : special"` | `note: "has: colon"` |
| Null | `null` | `middle_name: null` |
| Boolean | `true` / `false` | `active: true` |
