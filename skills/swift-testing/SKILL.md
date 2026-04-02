---
name: swift-testing
description: Swift Testing framework API reference — @Test, #expect, #require, @Suite, traits, parameterized tests, confirmation, and XCTest migration. Use when user asks about Swift Testing APIs or syntax. For generating test files, use generators-test-generator. For TDD workflows, use testing-tdd-feature.
---

> **First step:** Tell the user: "swift-testing skill loaded."

# Swift Testing Framework

The Swift Testing framework (`import Testing`) is Apple's modern test framework introduced in Xcode 16. It replaces XCTest's `XCTAssert` family with expressive macros (`#expect`, `#require`), supports parameterized tests, traits for conditional execution, and organizes tests with `@Suite` and `@Test` attributes.

## When This Skill Activates

- User asks about Swift Testing APIs (`@Test`, `#expect`, `#require`, `@Suite`, traits)
- User wants to migrate from XCTest to Swift Testing
- User asks about parameterized tests, confirmation, or test tags
- User wants to mix XCTest and Swift Testing in the same target
- User asks about the differences between Swift Testing and XCTest

**Do NOT activate** for these — use the specialized skill instead:
- Generating test boilerplate for existing code → `generators-test-generator`
- TDD workflow (red-green-refactor) → `testing-tdd-feature` or `testing-tdd-bug-fix`

## Decision Tree: Swift Testing vs XCTest

| Scenario | Use |
|----------|-----|
| New test target (Xcode 16+) | Swift Testing |
| UI tests (XCUIApplication) | XCTest (Swift Testing does not support UI testing) |
| Performance tests (measure blocks) | XCTest (Swift Testing has no performance API) |
| Objective-C test code | XCTest (Swift Testing is Swift-only) |
| Parameterized / data-driven tests | Swift Testing (first-class support) |
| Existing XCTest target, adding new tests | Either; they coexist in the same target |
| KIF or EarlGrey UI tests | XCTest |

## API Availability

| Requirement | Minimum |
|-------------|---------|
| Import | `import Testing` |
| Xcode | 16.0+ |
| Swift | 6.0+ |
| iOS deployment target | 16.0+ |
| macOS deployment target | 13.0+ |
| watchOS deployment target | 9.0+ |
| tvOS deployment target | 16.0+ |
| visionOS deployment target | 1.0+ |

> Swift Testing ships with the Xcode toolchain, not the OS SDK. Apps targeting iOS 16+ can use it as long as they build with Xcode 16+.

---

## Core APIs

### @Test and #expect

```swift
import Testing

@Test("Addition produces correct sum")
func addition() {
    let result = 2 + 3
    #expect(result == 5)
}
```

`#expect` captures the full expression on failure, showing both sides of the comparison automatically. No need for separate `assertEqual` / `assertTrue` variants.

### #require for Unwrapping and Preconditions

`#require` throws on failure, stopping the test immediately. Use it for preconditions and optional unwrapping.

```swift
@Test func loadUser() throws {
    let data = try #require(JSONLoader.load("user.json"))
    let user = try #require(User(data: data))
    #expect(user.name == "Alice")
}
```

### @Suite for Organization

```swift
@Suite("Authentication Tests")
struct AuthenticationTests {
    let service: AuthService

    init() {
        // Runs before each @Test (equivalent to setUp)
        service = AuthService(store: MockTokenStore())
    }

    @Test func loginSucceeds() async throws {
        let token = try await service.login(user: "admin", password: "secret")
        #expect(token.isValid)
    }

    @Test func loginFailsWithBadPassword() async throws {
        await #expect(throws: AuthError.invalidCredentials) {
            try await service.login(user: "admin", password: "wrong")
        }
    }
}
```

- `init()` replaces XCTest's `setUp()` -- runs before each test.
- `deinit` replaces `tearDown()` -- runs after each test (requires the suite to be a `class`).
- Suites can be nested structs for logical grouping.

---

## Traits

Traits modify test behavior. Pass them as arguments to `@Test` or `@Suite`.

### .enabled(if:) and .disabled()

```swift
@Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] != nil))
func onlyOnCI() {
    // Runs only in CI
}

@Test(.disabled("Blocked by #1234"))
func brokenFeature() {
    // Skipped with a reason
}
```

### .bug()

```swift
@Test(.bug("https://github.com/org/repo/issues/42", "Crash on empty input"))
func emptyInputHandling() {
    #expect(Parser.parse("") == .empty)
}
```

### .tags()

```swift
extension Tag {
    @Tag static var networking: Self
    @Tag static var database: Self
}

@Test(.tags(.networking))
func fetchData() async throws {
    let data = try await APIClient.shared.fetch("/items")
    #expect(!data.isEmpty)
}
```

Run only tagged tests from the command line:
```bash
swift test --filter .tags:networking
```

### .timeLimit()

```swift
@Test(.timeLimit(.minutes(1)))
func longRunningOperation() async throws {
    let result = try await processor.run(largeDataset)
    #expect(result.isComplete)
}
```

### Combining Traits

```swift
@Test(
    "Sync engine completes within time limit",
    .tags(.networking, .database),
    .timeLimit(.minutes(2)),
    .enabled(if: FeatureFlags.syncV2Enabled)
)
func syncEngine() async throws {
    // ...
}
```

---

## Parameterized Tests

### Basic Parameterized Test

```swift
@Test(arguments: ["hello", "world", "swift"])
func stringIsNotEmpty(_ value: String) {
    #expect(!value.isEmpty)
}
```

### Multiple Argument Collections

```swift
@Test(arguments: [1, 2, 3], ["a", "b", "c"])
func pairCombinations(number: Int, letter: String) {
    #expect(!letter.isEmpty)
    #expect(number > 0)
}
```

This produces the Cartesian product: (1,"a"), (1,"b"), (1,"c"), (2,"a"), ... (9 total).

### Zip Instead of Cartesian Product

```swift
@Test(arguments: zip([1, 2, 3], ["one", "two", "three"]))
func zippedPairs(number: Int, name: String) {
    #expect(!name.isEmpty)
}
```

### Enum-Based Arguments

```swift
enum Direction: CaseIterable {
    case north, south, east, west
}

@Test(arguments: Direction.allCases)
func directionHasOpposite(_ direction: Direction) {
    #expect(direction.opposite.opposite == direction)
}
```

### Custom Test Arguments with CustomTestStringConvertible

When test arguments are complex types, conform to `CustomTestStringConvertible` to get readable test names in Xcode's test navigator.

```swift
struct UserFixture: CustomTestStringConvertible, Sendable {
    let name: String
    let age: Int
    let expectedTier: MembershipTier

    var testDescription: String { "\(name) (age \(age))" }
}

@Test(arguments: [
    UserFixture(name: "Teen", age: 15, expectedTier: .junior),
    UserFixture(name: "Adult", age: 30, expectedTier: .standard),
    UserFixture(name: "Senior", age: 65, expectedTier: .senior),
])
func membershipTier(_ fixture: UserFixture) {
    let tier = MembershipTier.for(age: fixture.age)
    #expect(tier == fixture.expectedTier)
}
```

---

## Error Testing

### Expecting a Specific Error Type

```swift
@Test func invalidInputThrows() {
    #expect(throws: ValidationError.self) {
        try Validator.validate("")
    }
}
```

### Expecting a Specific Error Value

```swift
@Test func invalidInputThrowsSpecificError() {
    #expect(throws: ValidationError.emptyField("name")) {
        try Validator.validate(field: "name", value: "")
    }
}
```

### Inspecting the Thrown Error

```swift
@Test func errorContainsContext() throws {
    let error = try #require(throws: NetworkError.self) {
        try client.fetchSync(url: badURL)
    }
    #expect(error.statusCode == 404)
    #expect(error.url == badURL)
}
```

### Expecting No Error (the default)

```swift
// #expect does NOT throw -- this naturally fails if an error is thrown:
@Test func validInputSucceeds() throws {
    let result = try Validator.validate("hello")
    #expect(result.isValid)
}
```

---

## Async Test Support

Async tests work naturally. Mark the test function `async` and/or `throws`.

```swift
@Test func fetchUserProfile() async throws {
    let profile = try await api.fetchProfile(id: "user-1")
    #expect(profile.name == "Alice")
}
```

### Testing with Confirmation (replacing XCTestExpectation)

```swift
@Test func notificationPosted() async {
    await confirmation("UserDidLogin notification received") { confirm in
        let observer = NotificationCenter.default.addObserver(
            forName: .userDidLogin, object: nil, queue: .main
        ) { _ in
            confirm()
        }
        LoginManager.shared.login(user: "test", password: "test")
        NotificationCenter.default.removeObserver(observer)
    }
}
```

For multiple expected confirmations:

```swift
await confirmation("Progress callbacks", expectedCount: 3) { confirm in
    downloader.onProgress = { _ in confirm() }
    await downloader.download(url: fileURL)
}
```

---

## Migration from XCTest

### Assertion Mapping

| XCTest | Swift Testing |
|--------|--------------|
| `XCTAssertTrue(x)` | `#expect(x)` |
| `XCTAssertFalse(x)` | `#expect(!x)` |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertNotEqual(a, b)` | `#expect(a != b)` |
| `XCTAssertNil(x)` | `#expect(x == nil)` |
| `XCTAssertNotNil(x)` | `#expect(x != nil)` or `try #require(x)` |
| `XCTAssertGreaterThan(a, b)` | `#expect(a > b)` |
| `XCTAssertThrowsError(expr)` | `#expect(throws: SomeError.self) { expr }` |
| `XCTUnwrap(x)` | `try #require(x)` |
| `XCTFail("msg")` | `Issue.record("msg")` |

### Structure Mapping

| XCTest | Swift Testing |
|--------|--------------|
| `class FooTests: XCTestCase` | `@Suite struct FooTests` |
| `func testBar()` | `@Test func bar()` |
| `override func setUp()` | `init()` |
| `override func tearDown()` | `deinit` (class suites only) |
| `XCTestExpectation` + `wait(for:)` | `confirmation { }` |
| `addTeardownBlock { }` | `deinit` or explicit cleanup in test |

### Before (XCTest)

```swift
// ❌ XCTest style
import XCTest

class ParserTests: XCTestCase {
    var parser: Parser!

    override func setUp() {
        super.setUp()
        parser = Parser()
    }

    func testParseValidJSON() throws {
        let result = try parser.parse("{\"key\": \"value\"}")
        XCTAssertNotNil(result)
        XCTAssertEqual(result?["key"] as? String, "value")
    }

    func testParseInvalidJSONThrows() {
        XCTAssertThrowsError(try parser.parse("not json"))
    }
}
```

### After (Swift Testing)

```swift
// ✅ Swift Testing style
import Testing

@Suite struct ParserTests {
    let parser = Parser()

    @Test func parseValidJSON() throws {
        let result = try #require(parser.parse("{\"key\": \"value\"}"))
        #expect(result["key"] as? String == "value")
    }

    @Test func parseInvalidJSONThrows() {
        #expect(throws: ParserError.self) {
            try parser.parse("not json")
        }
    }
}
```

---

## Running Tests

### From Xcode

- **Cmd+U**: Run all tests
- Click the diamond next to a `@Test` or `@Suite` in the gutter
- Test Navigator (Cmd+6) shows Swift Testing and XCTest results together

### From the Command Line

```bash
# Run all tests
swift test

# Filter by test name
swift test --filter "ParserTests"

# Filter by tag
swift test --filter .tags:networking
```

---

## Mixing XCTest and Swift Testing

Swift Testing and XCTest coexist in the same test target. Rules:

1. Do **not** subclass `XCTestCase` in Swift Testing suites.
2. Do **not** use `@Test` on methods inside an `XCTestCase` subclass.
3. Each test file should use one framework consistently.
4. Both frameworks' tests appear together in Xcode's Test Navigator.

```swift
// File: ParserXCTests.swift -- XCTest
import XCTest

class ParserXCTests: XCTestCase {
    func testLegacyBehavior() {
        XCTAssertEqual(Parser.legacyMode, true)
    }
}

// File: ParserSwiftTests.swift -- Swift Testing
import Testing

@Suite struct ParserSwiftTests {
    @Test func modernBehavior() {
        #expect(Parser.modernMode == true)
    }
}
```

---

## Good and Bad Patterns

### Prefer #expect Over Boolean Assertions

```swift
// ❌ Bad: loses context on failure
#expect(result == true)
#expect(array.count == 3)

// ✅ Good: use the natural expression
#expect(result)
#expect(array.count == 3) // this is fine -- #expect shows both sides
```

### Prefer #require for Unwrapping

```swift
// ❌ Bad: force unwrap crashes the test runner
let user = users.first!
#expect(user.name == "Alice")

// ✅ Good: #require fails gracefully and stops the test
let user = try #require(users.first)
#expect(user.name == "Alice")
```

### Prefer Structs Over Classes for Suites

```swift
// ❌ Bad: unnecessary class when no tearDown is needed
@Suite class CalculatorTests {
    @Test func add() { #expect(Calculator.add(2, 3) == 5) }
}

// ✅ Good: struct with init for setup
@Suite struct CalculatorTests {
    let calculator = Calculator()
    @Test func add() { #expect(calculator.add(2, 3) == 5) }
}
```

### Use Parameterized Tests Instead of Copy-Paste

```swift
// ❌ Bad: duplicated tests
@Test func parseCSVComma() throws {
    let result = try CSVParser.parse("a,b,c")
    #expect(result == ["a", "b", "c"])
}
@Test func parseCSVSemicolon() throws {
    let result = try CSVParser.parse("a;b;c", delimiter: ";")
    #expect(result == ["a", "b", "c"])
}

// ✅ Good: parameterized
struct CSVCase: CustomTestStringConvertible, Sendable {
    let input: String
    let delimiter: Character
    let expected: [String]
    var testDescription: String { "delimiter: \(delimiter)" }
}

@Test(arguments: [
    CSVCase(input: "a,b,c", delimiter: ",", expected: ["a", "b", "c"]),
    CSVCase(input: "a;b;c", delimiter: ";", expected: ["a", "b", "c"]),
])
func parseCSV(_ testCase: CSVCase) throws {
    let result = try CSVParser.parse(testCase.input, delimiter: testCase.delimiter)
    #expect(result == testCase.expected)
}
```

### Use Traits for Conditional Tests

```swift
// ❌ Bad: runtime skip with guard
@Test func networkTest() async throws {
    guard ProcessInfo.processInfo.environment["CI"] != nil else { return }
    // ...
}

// ✅ Good: trait makes intent clear and shows "skipped" in results
@Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] != nil))
func networkTest() async throws {
    // ...
}
```
