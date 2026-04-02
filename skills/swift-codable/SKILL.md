---
name: swift-codable
description: Swift Codable patterns for JSON and TOON serialization — custom keys, containers, date strategies, polymorphism, and the toon-swift library. Use when working with Encodable, Decodable, JSONEncoder, JSONDecoder, TOONEncoder, TOONDecoder, or CodingKeys.
---

# Swift Codable & Serialization

Comprehensive patterns for encoding and decoding Swift types with `Codable`, covering JSON (Foundation) and TOON (toon-swift). Includes custom key mapping, manual container coding, date and data strategies, polymorphic types, and migration between formats.

## When This Skill Activates

- User is implementing `Codable`, `Encodable`, or `Decodable` conformance
- User needs custom `CodingKeys` or key strategies
- User is working with `JSONEncoder` / `JSONDecoder` configuration
- User is working with TOON format or the `toon-swift` / `ToonFormat` library
- User needs to decode nested, flattened, or polymorphic JSON/TOON
- User is debugging encoding/decoding errors
- User asks about date, data, or float encoding strategies
- User wants to reduce token usage in LLM prompts via TOON

## Decision Tree

```
What do you need?
├── Basic Codable conformance ──────────── See "Fundamentals" below
├── Custom key names or casing ─────────── See "Key Mapping Strategies"
├── Nested/flattened/wrapper decoding ──── See references/advanced-patterns.md
├── Polymorphic or heterogeneous types ─── See references/advanced-patterns.md
├── Date, Data, or Float strategies ────── See "Encoder/Decoder Configuration"
├── TOON format encoding/decoding ──────── See references/toon-format.md
├── Migrating JSON ↔ TOON ─────────────── See references/toon-format.md
└── Debugging decode failures ──────────── See "Debugging Codable Errors"
```

## Fundamentals

### Automatic Conformance

When all stored properties are `Codable`, the compiler synthesizes everything:

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String?
    let tags: [String]
    let isActive: Bool
}

// Encode
let data = try JSONEncoder().encode(user)

// Decode
let user = try JSONDecoder().decode(User.self, from: data)
```

Synthesis works for `struct`, `class`, `enum` (with raw values or associated values in Swift 5.5+), and `actor`.

### Codable Enums

```swift
// Raw-value enum — encodes as the raw value
enum Priority: String, Codable {
    case low, medium, high, critical
}

// Enum with associated values (Swift 5.5+)
enum PaymentMethod: Codable {
    case creditCard(last4: String, expiry: String)
    case bankTransfer(routingNumber: String)
    case applePay
}
```

## Key Mapping Strategies

### Custom CodingKeys

Map between Swift property names and serialized key names:

```swift
struct Article: Codable {
    let id: Int
    let title: String
    let bodyText: String
    let publishedAt: Date
    let authorName: String

    enum CodingKeys: String, CodingKey {
        case id
        case title
        case bodyText = "body_text"
        case publishedAt = "published_at"
        case authorName = "author_name"
    }
}
```

### Automatic Key Strategy

Convert all keys without manual `CodingKeys`:

```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase  // body_text → bodyText

let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase    // bodyText → body_text
```

Custom key conversion:

```swift
decoder.keyDecodingStrategy = .custom { codingPath in
    let lastKey = codingPath.last!.stringValue
    // e.g., strip a prefix
    let stripped = lastKey.hasPrefix("_") ? String(lastKey.dropFirst()) : lastKey
    return AnyCodingKey(stringValue: stripped)!
}
```

## Encoder/Decoder Configuration

### Date Strategies

```swift
let encoder = JSONEncoder()

// ISO 8601 (recommended for APIs)
encoder.dateEncodingStrategy = .iso8601

// Unix timestamp (seconds since epoch)
encoder.dateEncodingStrategy = .secondsSince1970

// Custom format
let formatter = DateFormatter()
formatter.dateFormat = "yyyy-MM-dd"
formatter.locale = Locale(identifier: "en_US_POSIX")
encoder.dateEncodingStrategy = .formatted(formatter)

// Full control
encoder.dateEncodingStrategy = .custom { date, encoder in
    var container = encoder.singleValueContainer()
    try container.encode(date.timeIntervalSince1970 * 1000) // milliseconds
}
```

The same strategies are available on `JSONDecoder` via `dateDecodingStrategy`.

### Data Strategies

```swift
encoder.dataEncodingStrategy = .base64        // default
encoder.dataEncodingStrategy = .deferredToData // raw bytes as [UInt8] array
encoder.dataEncodingStrategy = .custom { data, encoder in
    var container = encoder.singleValueContainer()
    try container.encode(data.map { String(format: "%02x", $0) }.joined())
}
```

### Non-Conforming Float Strategies

```swift
// Handle Infinity and NaN
decoder.nonConformingFloatDecodingStrategy = .convertFromString(
    positiveInfinity: "Infinity",
    negativeInfinity: "-Infinity",
    nan: "NaN"
)
```

### Output Formatting

```swift
encoder.outputFormatting = [.prettyPrinted, .sortedKeys, .withoutEscapingSlashes]
```

## Debugging Codable Errors

### DecodingError Cases

```swift
do {
    let model = try JSONDecoder().decode(MyModel.self, from: data)
} catch let DecodingError.keyNotFound(key, context) {
    print("Missing key: \(key.stringValue)")
    print("Path: \(context.codingPath.map(\.stringValue).joined(separator: "."))")
} catch let DecodingError.typeMismatch(type, context) {
    print("Expected \(type) at \(context.codingPath.map(\.stringValue).joined(separator: "."))")
    print("Debug: \(context.debugDescription)")
} catch let DecodingError.valueNotFound(type, context) {
    print("Null value for non-optional \(type) at \(context.codingPath.map(\.stringValue).joined(separator: "."))")
} catch let DecodingError.dataCorrupted(context) {
    print("Corrupted data at \(context.codingPath.map(\.stringValue).joined(separator: "."))")
    print("Debug: \(context.debugDescription)")
}
```

### Common Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `keyNotFound` | JSON uses `snake_case`, Swift uses `camelCase` | Add `.convertFromSnakeCase` or custom `CodingKeys` |
| `typeMismatch` for `Int` | JSON value is `"42"` (string) | Decode as `String` and convert, or use a custom `init(from:)` |
| `typeMismatch` for `Bool` | API sends `0`/`1` instead of `true`/`false` | Custom decode with `Int` fallback |
| `valueNotFound` | Null in JSON for non-optional property | Make the property optional or provide a `decodeIfPresent` default |
| `dataCorrupted` for `Date` | Date format doesn't match strategy | Check the strategy matches the API's format exactly |
| Silent empty results | Wrong root type (object vs. array) | Verify the top-level JSON structure |

### Providing Defaults for Missing Keys

```swift
struct Settings: Decodable {
    let theme: String
    let fontSize: Int
    let notifications: Bool

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        theme = try container.decodeIfPresent(String.self, forKey: .theme) ?? "system"
        fontSize = try container.decodeIfPresent(Int.self, forKey: .fontSize) ?? 14
        notifications = try container.decodeIfPresent(Bool.self, forKey: .notifications) ?? true
    }

    enum CodingKeys: String, CodingKey {
        case theme, fontSize, notifications
    }
}
```

## References

- **[references/advanced-patterns.md](references/advanced-patterns.md)** — Manual container coding, nested key decoding, flattened structures, polymorphic types, type erasure, and wrapper patterns
- **[references/toon-format.md](references/toon-format.md)** — TOON format overview, `TOONEncoder`/`TOONDecoder` usage, configuration, tabular arrays, key folding, and JSON-to-TOON migration
- Related skills: `generators-networking-layer` (API client with Codable), `swiftdata-patterns` (model layer)
