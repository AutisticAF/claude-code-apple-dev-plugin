---
name: swift-regex-builder
description: Swift Regex builder DSL for type-safe pattern matching, captures, quantifiers, and Foundation parsers. Use when building complex regex patterns in Swift.
---

# Swift Regex Builder

Type-safe regular expressions using Swift's `RegexBuilder` DSL. Covers `Regex` literals, the builder DSL, captures, quantifiers, character classes, anchors, Foundation parsers, and migration from `NSRegularExpression`.

## When This Skill Activates

Use this skill when the user:
- Asks about **Regex** or **RegexBuilder** in Swift
- Wants to **parse**, **match**, or **extract** data from strings using patterns
- Mentions **captures**, **quantifiers**, or **character classes** in Swift
- Asks about **/regex literal/** syntax in Swift
- Wants to use **Foundation parsers** (dates, numbers, currency) inside regex
- Asks about **replacing**, **splitting**, or **trimming** with regex
- Wants to **migrate from NSRegularExpression** to modern Swift regex
- Mentions `firstMatch(of:)`, `wholeMatch(of:)`, or `matches(of:)`

## Decision Tree

```
What kind of pattern matching do you need?
|
+-- Simple, short pattern (e.g., email, hex color)
|   --> Regex literal: /pattern/
|
+-- Complex pattern with structured data extraction
|   |
|   +-- Need Foundation parsers (dates, numbers, currency)
|   |   --> RegexBuilder DSL with parser components
|   |
|   +-- Need readable, composable, multi-line pattern
|       --> RegexBuilder DSL
|
+-- Dynamic pattern built from user input at runtime
|   --> Regex(String) initializer (throws)
|
+-- Must support iOS 15 or earlier
    --> NSRegularExpression (legacy)
```

## API Availability

| API | Minimum Version | Notes |
|-----|----------------|-------|
| `Regex` type | Swift 5.7 / iOS 16 / macOS 13 | Core regex type |
| Regex literals `/pattern/` | Swift 5.7 / iOS 16 / macOS 13 | Compiler-checked at build time |
| `RegexBuilder` DSL | Swift 5.7 / iOS 16 / macOS 13 | `import RegexBuilder` |
| `firstMatch(of:)` | Swift 5.7 / iOS 16 / macOS 13 | First match in string |
| `wholeMatch(of:)` | Swift 5.7 / iOS 16 / macOS 13 | Entire string must match |
| `matches(of:)` | Swift 5.7 / iOS 16 / macOS 13 | All non-overlapping matches |
| `Regex(String)` | Swift 5.7 / iOS 16 / macOS 13 | Runtime pattern, throws on invalid |
| Foundation parsers in regex | Swift 5.7 / iOS 16 / macOS 13 | `.localizedInteger`, `.iso8601`, etc. |

## Basic Regex Type and Matching

```swift
let pattern = /\d{3}-\d{4}/
let input = "Call 555-1234 or 555-5678"

if let match = input.firstMatch(of: pattern) { print(match.output) }  // "555-1234"
if let match = input.wholeMatch(of: /Call .+/) { print(match.output) } // full string
let all = input.matches(of: pattern)  // all non-overlapping matches
let has = input.contains(pattern)     // Bool
if let m = "555-1234 etc".prefixMatch(of: pattern) { print(m.output) } // prefix only
```

## Regex Literals

```swift
// Compiler-checked regex literals
let hexColor = /#[0-9a-fA-F]{6}/
let email = /[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/

// Extended literal with comments and whitespace (use #/.../#)
let date = #/
    (?<year>\d{4})   # Year
    -
    (?<month>\d{2})  # Month
    -
    (?<day>\d{2})    # Day
/#

if let match = "2025-12-31".wholeMatch(of: date) {
    print(match.year)  // "2025"
    print(match.month) // "12"
    print(match.day)   // "31"
}
```

## RegexBuilder DSL

```swift
import RegexBuilder

// Basic builder syntax
let phonePattern = Regex {
    Repeat(count: 3) { .digit }
    "-"
    Repeat(count: 4) { .digit }
}

// Composable: assign sub-patterns to variables
let areaCode = Regex {
    "("
    Repeat(count: 3) { .digit }
    ") "
}

let fullPhone = Regex {
    areaCode
    phonePattern
}

if let m = "(415) 555-1234".wholeMatch(of: fullPhone) {
    print(m.output) // "(415) 555-1234"
}
```

## Components: Quantifiers, Choices, and Captures

```swift
import RegexBuilder

// One: exactly one occurrence (default, often implicit)
Regex { One(.digit) }

// OneOrMore: one or more
Regex { OneOrMore(.word) }

// ZeroOrMore: zero or more
Regex { ZeroOrMore(.whitespace) }

// Optionally: zero or one
Regex {
    Optionally { "-" }
    OneOrMore(.digit)
}

// Repeat: exact count or range
Regex { Repeat(count: 3) { .digit } }
Regex { Repeat(2...4) { .digit } }

// ChoiceOf: alternation (like | in regex)
let scheme = Regex {
    ChoiceOf {
        "http"
        "https"
        "ftp"
    }
    "://"
}

// Capture: extract matched content
let csvRow = Regex {
    Capture { OneOrMore(.word) }       // column 1
    ","
    Capture { OneOrMore(.digit) }      // column 2
}

if let match = "Alice,42".wholeMatch(of: csvRow) {
    let (whole, name, age) = match.output
    print(name) // "Alice"
    print(age)  // "42"
}

// TryCapture: transform or validate during capture
let intCapture = Regex {
    TryCapture {
        OneOrMore(.digit)
    } transform: { substring in
        Int(substring)
    }
}

if let match = "score:99".firstMatch(of: Regex {
    "score:"
    intCapture
}) {
    let value: Int = match.output.1  // 99, already an Int
}
```

## Character Classes and Anchors

```swift
import RegexBuilder

// Built-in: .digit (\d), .word (\w), .whitespace (\s), .any (.), .anyNonNewline
let vowel = Regex { One(.anyOf("aeiouAEIOU")) }
let hexDigit = Regex {
    One { CharacterClass(.digit, ("a"..."f"), ("A"..."F")) }
}
let nonDigit = CharacterClass.digit.inverted

// Anchors
Regex { Anchor.startOfLine; OneOrMore(.word) }
Regex { OneOrMore(.word); Anchor.endOfLine }
Regex { Anchor.wordBoundary; "swift"; Anchor.wordBoundary }
```

## Foundation Parsers in Regex

Foundation types conform to `CustomConsumingRegexComponent` for locale-aware parsing inside regex.

```swift
import RegexBuilder

// Localized integer (handles grouping separators)
let priceTag = Regex { "$"; Capture { .localizedInteger } }
if let match = "$1,234".firstMatch(of: priceTag) {
    let amount: Int = match.output.1  // 1234
}

// Date parsing with strategy
let dateRegex = Regex {
    "Due: "
    Capture { One(.date(.numeric, locale: Locale(identifier: "en_US"), timeZone: .gmt)) }
}

// ISO 8601, floating-point, currency
Regex { Capture { .iso8601 } }
Regex { Capture { .localizedDouble }; "F" }
Regex { Capture { .localizedCurrency(code: "USD") } }
```

## Named Captures and Output Tuples

```swift
import RegexBuilder

// Reference captures for complex patterns
let timestamp = Reference(Substring.self)
let level = Reference(Substring.self)
let message = Reference(Substring.self)

let logPattern = Regex {
    "["
    Capture(as: timestamp) { OneOrMore(.any, .reluctant) }
    "] "
    Capture(as: level) { ChoiceOf { "INFO"; "WARN"; "ERROR" } }
    ": "
    Capture(as: message) { OneOrMore(.any) }
}

if let match = "[2025-01-15 09:30] ERROR: Disk full".firstMatch(of: logPattern) {
    print(match[timestamp]) // "2025-01-15 09:30"
    print(match[level])     // "ERROR"
    print(match[message])   // "Disk full"
}

// Output tuple with TryCapture transforms
let coordinate = Regex {
    TryCapture { OneOrMore(.any, .reluctant) } transform: { Double(String($0)) }
    ","
    TryCapture { OneOrMore(.any) } transform: { Double(String($0)) }
}
// match.output is (Substring, Double, Double)
```

## String Processing: Replacing, Splitting, Trimming

```swift
var text = "Hello World"
text.replace(/World/, with: "Swift")                      // "Hello Swift"
"a, b, c".replacing(/,\s*/, with: "|")                    // "a|b|c"
"one::two::three".split(separator: /::/)                   // ["one", "two", "three"]
"  hello  ".trimmingPrefix(/\s+/)                          // "hello  "
"v2_rc1".replacing(/\d+/) { match in match.output + "0" }  // closure-based replace
```

## Performance: Compile Once, Reuse

```swift
// Store as a static or instance property, not inside a loop
struct LogParser {
    // Compiled once
    private static let pattern = Regex {
        "["
        Capture { OneOrMore(.any, .reluctant) }
        "] "
        Capture { OneOrMore(.any) }
    }

    func parse(_ line: String) -> (timestamp: Substring, message: Substring)? {
        guard let match = line.firstMatch(of: Self.pattern) else { return nil }
        return (match.output.1, match.output.2)
    }
}
```

## Migration from NSRegularExpression

```swift
// Before: NSRegularExpression (verbose, stringly typed)
let nsRegex = try NSRegularExpression(pattern: "\\d{3}-\\d{4}")
let nsRange = NSRange(input.startIndex..., in: input)
for nsMatch in nsRegex.matches(in: input, range: nsRange) {
    if let range = Range(nsMatch.range, in: input) { print(input[range]) }
}

// After: Swift Regex (concise, type-safe)
for match in input.matches(of: /\d{3}-\d{4}/) { print(match.output) }
```

## Good and Bad Patterns

```swift
// RegexBuilder for complex patterns with Foundation parsers
import RegexBuilder
let invoiceLine = Regex {
    Capture { OneOrMore(.word) }
    /\s+/
    Capture { .localizedInteger }
    " x $"
    Capture { .localizedDouble }
}

// Regex literals for simple, short patterns
let digits = /\d+/
let tag = /<[^>]+>/
```

```swift
// Compile once, reuse
struct Validator {
    private static let emailRegex = /[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/
    func isValid(_ email: String) -> Bool {
        email.wholeMatch(of: Self.emailRegex) != nil
    }
}
```

```swift
// Avoid string concatenation for regex patterns
let pattern = "\\d{" + String(count) + "}"               // fragile, no compile-time checks
let regex = Regex { Repeat(count) { .digit } }            // type-safe alternative

// Avoid NSRegularExpression when targeting iOS 16+
let nsRegex = try NSRegularExpression(pattern: "(\\d+)")  // verbose, NSRange conversion
let swiftRegex = /(\d+)/                                   // concise, type-safe
```
