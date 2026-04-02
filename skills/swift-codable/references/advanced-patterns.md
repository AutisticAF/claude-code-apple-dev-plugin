# Advanced Codable Patterns

## Manual Container Coding

When automatic synthesis isn't enough, implement `init(from:)` and `encode(to:)` manually using keyed, unkeyed, and single-value containers.

### Keyed Containers

```swift
struct Event: Codable {
    let id: String
    let name: String
    let startDate: Date
    let endDate: Date
    let metadata: [String: String]

    enum CodingKeys: String, CodingKey {
        case id, name, startDate = "start_date", endDate = "end_date", metadata
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(String.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)
        startDate = try container.decode(Date.self, forKey: .startDate)
        endDate = try container.decode(Date.self, forKey: .endDate)
        metadata = try container.decodeIfPresent([String: String].self, forKey: .metadata) ?? [:]
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(name, forKey: .name)
        try container.encode(startDate, forKey: .startDate)
        try container.encode(endDate, forKey: .endDate)
        if !metadata.isEmpty {
            try container.encode(metadata, forKey: .metadata)
        }
    }
}
```

### Unkeyed Containers (Arrays)

Decode heterogeneous arrays by attempting types in order:

```swift
struct CoordinatePath: Decodable {
    let points: [[Double]]

    init(from decoder: Decoder) throws {
        var container = try decoder.unkeyedContainer()
        var result: [[Double]] = []
        while !container.isAtEnd {
            var nested = try container.nestedUnkeyedContainer()
            var point: [Double] = []
            while !nested.isAtEnd {
                point.append(try nested.decode(Double.self))
            }
            result.append(point)
        }
        points = result
    }
}
```

### Single-Value Containers

Wrap or transform a single encoded representation:

```swift
struct Identifier: Codable {
    let value: String

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        // Accept both String and Int representations
        if let stringValue = try? container.decode(String.self) {
            value = stringValue
        } else {
            let intValue = try container.decode(Int.self)
            value = String(intValue)
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(value)
    }
}
```

## Nested Key Decoding

### Decoding from Nested Objects

Flatten a nested JSON structure into a flat Swift type:

```json
{
    "id": 1,
    "name": "Ada",
    "address": {
        "street": "123 Main St",
        "city": "London",
        "country": "UK"
    }
}
```

```swift
struct Contact: Decodable {
    let id: Int
    let name: String
    let street: String
    let city: String
    let country: String

    enum CodingKeys: String, CodingKey {
        case id, name, address
    }

    enum AddressKeys: String, CodingKey {
        case street, city, country
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(Int.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)

        let address = try container.nestedContainer(keyedBy: AddressKeys.self, forKey: .address)
        street = try address.decode(String.self, forKey: .street)
        city = try address.decode(String.self, forKey: .city)
        country = try address.decode(String.self, forKey: .country)
    }
}
```

### Encoding to Nested Objects

Group flat Swift properties into nested structure on output:

```swift
struct Contact: Codable {
    let id: Int
    let name: String
    let street: String
    let city: String
    let country: String

    enum CodingKeys: String, CodingKey {
        case id, name, address
    }

    enum AddressKeys: String, CodingKey {
        case street, city, country
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(name, forKey: .name)

        var address = container.nestedContainer(keyedBy: AddressKeys.self, forKey: .address)
        try address.encode(street, forKey: .street)
        try address.encode(city, forKey: .city)
        try address.encode(country, forKey: .country)
    }
}
```

## Polymorphic / Heterogeneous Types

### Discriminated Unions (Type Field)

When a JSON array contains objects with a `type` field:

```json
[
    {"type": "text", "content": "Hello"},
    {"type": "image", "url": "https://example.com/photo.jpg", "width": 800},
    {"type": "divider"}
]
```

```swift
enum Block: Decodable {
    case text(content: String)
    case image(url: URL, width: Int)
    case divider

    enum CodingKeys: String, CodingKey {
        case type, content, url, width
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let type = try container.decode(String.self, forKey: .type)

        switch type {
        case "text":
            let content = try container.decode(String.self, forKey: .content)
            self = .text(content: content)
        case "image":
            let url = try container.decode(URL.self, forKey: .url)
            let width = try container.decode(Int.self, forKey: .width)
            self = .image(url: url, width: width)
        case "divider":
            self = .divider
        default:
            throw DecodingError.dataCorrupted(
                DecodingError.Context(
                    codingPath: decoder.codingPath,
                    debugDescription: "Unknown block type: \(type)"
                )
            )
        }
    }
}
```

### Encodable Polymorphism

```swift
extension Block: Encodable {
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)

        switch self {
        case .text(let content):
            try container.encode("text", forKey: .type)
            try container.encode(content, forKey: .content)
        case .image(let url, let width):
            try container.encode("image", forKey: .type)
            try container.encode(url, forKey: .url)
            try container.encode(width, forKey: .width)
        case .divider:
            try container.encode("divider", forKey: .type)
        }
    }
}
```

### Try-Each-Type Pattern

When there's no discriminator field, attempt decoding each variant:

```swift
enum SearchResult: Decodable {
    case user(User)
    case product(Product)
    case article(Article)

    init(from decoder: Decoder) throws {
        if let user = try? User(from: decoder) {
            self = .user(user)
        } else if let product = try? Product(from: decoder) {
            self = .product(product)
        } else {
            self = .article(try Article(from: decoder))
        }
    }
}
```

This is less efficient than discriminated unions — use only when the API doesn't provide a type field.

## Wrapper and Pagination Patterns

### Unwrapping API Envelopes

```json
{
    "status": "ok",
    "data": { "id": 1, "name": "Ada" },
    "meta": { "request_id": "abc-123" }
}
```

```swift
struct APIResponse<T: Decodable>: Decodable {
    let status: String
    let data: T
    let meta: Meta?

    struct Meta: Decodable {
        let requestId: String
    }
}

// Usage
let response = try decoder.decode(APIResponse<User>.self, from: data)
let user = response.data
```

### Paginated Responses

```swift
struct Page<T: Decodable>: Decodable {
    let items: [T]
    let totalCount: Int
    let nextCursor: String?

    enum CodingKeys: String, CodingKey {
        case items, totalCount = "total_count", nextCursor = "next_cursor"
    }
}
```

## Dynamic Keys

When JSON keys are data rather than a fixed schema:

```json
{
    "2025-01-01": {"temperature": 5.2, "humidity": 80},
    "2025-01-02": {"temperature": 6.1, "humidity": 75}
}
```

```swift
struct DynamicCodingKey: CodingKey {
    var stringValue: String
    var intValue: Int?

    init?(stringValue: String) { self.stringValue = stringValue }
    init?(intValue: Int) { self.stringValue = String(intValue); self.intValue = intValue }
}

struct WeatherLog: Decodable {
    let entries: [String: WeatherEntry]

    struct WeatherEntry: Decodable {
        let temperature: Double
        let humidity: Int
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: DynamicCodingKey.self)
        var result: [String: WeatherEntry] = [:]
        for key in container.allKeys {
            result[key.stringValue] = try container.decode(WeatherEntry.self, forKey: key)
        }
        entries = result
    }
}
```

## Lossy Collections

Decode arrays that may contain malformed elements, skipping failures:

```swift
struct LossyArray<Element: Decodable>: Decodable {
    let elements: [Element]

    init(from decoder: Decoder) throws {
        var container = try decoder.unkeyedContainer()
        var result: [Element] = []
        while !container.isAtEnd {
            if let element = try? container.decode(Element.self) {
                result.append(element)
            } else {
                _ = try? container.decode(AnyCodableValue.self) // advance past the bad element
            }
        }
        elements = result
    }
}

// Minimal type to skip any JSON value
private struct AnyCodableValue: Decodable {}
```

## Computed and Transient Properties

Properties that shouldn't be serialized:

```swift
struct Product: Codable {
    let price: Double
    let quantity: Int

    // Not included in coding — computed on access
    var total: Double { price * Double(quantity) }

    // Excluded by listing only desired keys in CodingKeys
    var isInCart: Bool = false

    enum CodingKeys: String, CodingKey {
        case price, quantity
        // isInCart intentionally omitted
    }
}
```

## Property Wrappers for Codable

Reusable decoding strategies via property wrappers:

```swift
@propertyWrapper
struct DefaultEmpty<T: Decodable & RangeReplaceableCollection>: Decodable {
    var wrappedValue: T

    init(wrappedValue: T = T()) {
        self.wrappedValue = wrappedValue
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        wrappedValue = (try? container.decode(T.self)) ?? T()
    }
}

@propertyWrapper
struct DefaultFalse: Decodable {
    var wrappedValue: Bool

    init(wrappedValue: Bool = false) {
        self.wrappedValue = wrappedValue
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        wrappedValue = (try? container.decode(Bool.self)) ?? false
    }
}

// Usage
struct UserProfile: Decodable {
    let name: String
    @DefaultEmpty var tags: [String]
    @DefaultFalse var isVerified: Bool
}
```

Note: Property wrappers require `KeyedDecodingContainer` extensions for synthesis to work with optional keys. For simpler cases, prefer explicit `decodeIfPresent` with `??` in a manual `init(from:)`.
