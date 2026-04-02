---
name: swift-concurrency-patterns
description: Swift concurrency patterns including Swift 6.2 approachable concurrency (@concurrent, isolated conformances, default MainActor inference), structured concurrency, actors, continuations, and migration. Use when reviewing or building async code, fixing data race errors, adopting Swift 6.2 concurrency features, or migrating to Swift 6.
---

# Swift Concurrency Patterns

Comprehensive guide for Swift concurrency covering async/await, structured concurrency, actors, and the Swift 6.2 "Approachable Concurrency" features. Focuses on patterns that prevent data races and common mistakes that cause crashes.

## When This Skill Activates

- User has data race errors or actor isolation compiler errors
- User is migrating to Swift 6 strict concurrency
- User asks about async/await, actors, Sendable, TaskGroup, or MainActor
- User needs to bridge legacy completion-handler APIs to async/await
- User is working with Swift 6.2 features (@concurrent, isolated conformances)
- User asks about default MainActor inference or the "infer main actor" build setting
- User wants to migrate from Swift 6.0/6.1 strict concurrency to 6.2
- User asks how async functions behave differently in Swift 6.2
- User has concurrency bugs (actor reentrancy, task cancellation, UI freezes)

## Decision Tree

```
What concurrency problem are you solving?
│
├─ Swift 6 compiler errors / migration
│  └─ references/migration-guide.md
│
├─ Swift 6.2 new features (@concurrent, isolated conformances)
│  └─ references/swift62-concurrency.md
│
├─ Running work in parallel (async let, TaskGroup)
│  └─ references/structured-concurrency.md
│
├─ Thread safety for shared mutable state
│  └─ references/actors-and-isolation.md
│
├─ Bridging old APIs (delegates, callbacks) to async/await
│  └─ references/continuations-bridging.md
│
└─ General async/await patterns
   └─ See macos-coding-best-practices/references/modern-concurrency.md for basics
```

## Quick Reference

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| `async let` | Fixed number of parallel operations | `references/structured-concurrency.md` |
| `withTaskGroup` | Dynamic number of parallel operations | `references/structured-concurrency.md` |
| `withDiscardingTaskGroup` | Fire-and-forget parallel operations | `references/structured-concurrency.md` |
| `.task { }` modifier | Load data when view appears | `references/structured-concurrency.md` |
| `.task(id:)` modifier | Re-load when a value changes | `references/structured-concurrency.md` |
| `actor` | Shared mutable state protection | `references/actors-and-isolation.md` |
| `@MainActor` | UI-bound state and updates | `references/actors-and-isolation.md` |
| `@concurrent` | Explicitly offload to background (6.2) | `references/swift62-concurrency.md` |
| Isolated conformances | `@MainActor` type conforming to protocol (6.2) | `references/swift62-concurrency.md` |
| `withCheckedContinuation` | Bridge callback API to async | `references/continuations-bridging.md` |
| `AsyncStream` | Bridge delegate/notification API to async sequence | `references/continuations-bridging.md` |
| Strict concurrency migration | Incremental Swift 6 adoption | `references/migration-guide.md` |

## Process

### 1. Identify the Problem

Read the user's code or error messages to determine:
- Is this a compiler error (strict concurrency) or a runtime issue (data race, crash)?
- What Swift version and concurrency checking level are they using?
- Are they migrating existing code or writing new code?

### 2. Load Relevant Reference Files

Based on the problem, read from this directory:
- `references/swift62-concurrency.md` — Swift 6.2 approachable concurrency features
- `references/structured-concurrency.md` — async let, TaskGroup, .task modifier lifecycle
- `references/actors-and-isolation.md` — Actor patterns, reentrancy, @MainActor, Sendable
- `references/continuations-bridging.md` — withCheckedContinuation, AsyncStream, legacy bridging
- `references/migration-guide.md` — Incremental Swift 6 strict concurrency adoption

### 3. Review Checklist

- [ ] No blocking calls on `@MainActor` (use `await` for long operations)
- [ ] Shared mutable state protected by an actor (not locks or DispatchQueue)
- [ ] `Sendable` conformance correct for types crossing isolation boundaries
- [ ] Task cancellation handled (check `Task.isCancelled` or `Task.checkCancellation()`)
- [ ] No unstructured `Task {}` where structured concurrency (`.task`, `TaskGroup`) would work
- [ ] Actor reentrancy considered at suspension points
- [ ] `withCheckedContinuation` called exactly once (not zero, not twice)
- [ ] `.task(id:)` used instead of manual `onChange` + cancel patterns

### 4. Cross-Reference

- For **async/await basics and actor fundamentals**, see `macos-coding-best-practices/references/modern-concurrency.md`
- For **networking concurrency patterns**, see `generators-networking-layer/references/networking-patterns.md`
- For **SwiftData concurrency** (@ModelActor), see `macos-swiftdata-architecture/references/repository-pattern.md`
- For **auth token refresh with actors**, see `generators-auth-flow/references/auth-patterns.md`

## References

- [Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Migrating to Swift 6](https://www.swift.org/migration/documentation/migrationguide/)
- Apple doc: [Swift Concurrency Updates](https://developer.apple.com/documentation/swift/concurrency)
